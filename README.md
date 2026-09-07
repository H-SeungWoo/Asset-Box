# AssetBox

**Unity·Unreal 개발자와 테크니컬 아티스트(TA)가 3D 에셋을 공유·관리하는 웹 서비스**

모델과 텍스처를 묶어 등록하고, 필요한 에셋을 찾아 내려받거나 제작을 요청할 수 있도록 개발했습니다. 이 저장소는 백엔드 코드이며, 아래는 **한승우가 담당한 파일 처리 기능**을 중심으로 정리했습니다.

[서비스 바로가기](https://assetbox.cloud/) · [API 계약 문서](./docs/asset-post-api.md) · [배포 가이드](./DEPLOY.md)

| 구분 | 내용 |
| --- | --- |
| 개발 기간 | 2026.05.25 ~ 2026.07.12 |
| 팀 규모 | 8인 팀 프로젝트 |
| 담당 역할 | 백엔드 — 파일 검증·업로드, AWS S3 저장·삭제 흐름 / TA 요구사항 조율 |
| 핵심 기술 | Java, Spring Boot, JPA, MySQL, AWS S3, Docker |

### 먼저 볼 구현 3가지

| 구현 | 해결한 문제 | 대표 파일 |
| --- | --- | --- |
| **ZIP 검증·압축 해제** | 모델과 텍스처를 분류하고, 잘못된 경로·과도한 용량을 검사 | [ZipExtractService.java](./src/main/java/io/teabag/assetbox/file/service/ZipExtractService.java) |
| **업로드 실패 보상** | 업로드 도중 실패했을 때 이미 저장한 S3 객체 삭제 | [FileServiceImpl.java](./src/main/java/io/teabag/assetbox/file/service/FileServiceImpl.java) |
| **7일 유예 후 삭제** | 삭제 요청과 실제 파일 제거를 분리하고, 실패한 삭제는 다음 배치에서 재시도 | [StoragePurgeService.java](./src/main/java/io/teabag/assetbox/file/service/StoragePurgeService.java) |

## 담당 역할

- TA 과정 수강생들과 초기 요구사항 인터뷰부터 중간 검토 브리핑까지 진행하며, 실제 에셋 제작·공유 과정에 필요한 기능과 개발 범위를 구체화했습니다.
- 모델·텍스처 ZIP의 업로드와 검증, S3 저장, 파일 메타데이터 관리를 구현했습니다.
- 업로드 실패 시 보상 삭제와 삭제 요청 후 유예 기간을 두는 파일 수명 주기를 구현했습니다.

요구사항 논의 과정은 [TA 첫 회의](./docs/09_meetings/11-05-2026_TA팀_첫회의.md)와 [중간 프로토타입 발표](./docs/09_meetings/22-05-2026_TA팀_프로토타입발표.md)에 정리되어 있습니다.

## 1. ZIP 하나로 원본 다운로드와 미리보기용 파일 준비

3D 에셋은 모델뿐 아니라 텍스처와 폴더 구조도 함께 전달해야 합니다. 원본 ZIP은 다운로드용으로 보관하고, 압축을 해제한 모델과 텍스처는 웹 미리보기에서 사용할 수 있도록 별도로 저장했습니다.

```text
ZIP 업로드
  → 임시 파일 저장 / 원본 ZIP을 S3에 저장
  → ZIP 내부 경로·확장자·개수·용량 검사 및 압축 해제
  → 모델·텍스처를 구분해 S3에 저장 / DB에 메타데이터 저장
  → 다운로드용 원본과 미리보기용 파일 정보를 구분해 반환
```

### 경로 검사: 지정된 폴더 안에서만 압축 해제

ZIP 내부 경로를 정규화한 뒤 작업 폴더를 벗어나는지 확인합니다. 서로 다른 표기로 같은 위치를 가리키는 경우에도 중복 경로로 거부합니다.

```java
Path targetPath = normalizedExtractDir.resolve(normalizedEntryName).normalize();
if (!targetPath.startsWith(normalizedExtractDir)) {
    throw new BusinessException(ErrorCode.ZIP_INVALID);
}

if (!extractedPaths.add(targetPath)) {
    throw new BusinessException(ErrorCode.ZIP_INVALID);
}
```

[원본 코드 — ZipExtractService.java, 경로·중복 검사](https://github.com/H-SeungWoo/Asset-Box/blob/2bea504e71647dd744062e038e4fa01469efc91a/src/main/java/io/teabag/assetbox/file/service/ZipExtractService.java#L73-L80)

### 용량 검사: 실제 압축 해제량을 읽는 동안 제한

압축 파일의 크기만 확인하지 않고 스트림에서 읽은 바이트를 누적해 개별 파일과 전체 압축 해제 용량을 제한합니다.

```java
while ((read = zipInputStream.read(buffer)) != -1) {
    extractedSize += read;
    if (extractedSize > maxExtractedFileSizeBytes) {
        throw new BusinessException(ErrorCode.SIZE_INVALID);
    }

    if (currentTotalSize + extractedSize > maxExtractedTotalSizeBytes) {
        throw new BusinessException(ErrorCode.FILE_TOTAL_SIZE_INVALID);
    }

    outputStream.write(buffer, 0, read);
}
```

[원본 코드 — ZipExtractService.java, 스트림 복사·용량 검사](https://github.com/H-SeungWoo/Asset-Box/blob/2bea504e71647dd744062e038e4fa01469efc91a/src/main/java/io/teabag/assetbox/file/service/ZipExtractService.java#L133-L158)

현재 ZIP 검증은 파일 수 최대 100개, 개별·전체 압축 해제 용량 각각 50 MiB를 기준으로 합니다. 모델은 FBX 또는 GLB 한 개를 허용하며, 모델이 없거나 여러 개인 ZIP은 거부합니다. 중첩된 텍스처 폴더의 상대 경로는 보존합니다.

## 2. 업로드 도중 실패하면 이미 저장한 S3 객체 보상 삭제

DB 트랜잭션이 롤백되어도 S3에 저장한 파일은 자동으로 지워지지 않습니다. ZIP 업로드 과정에서 저장에 성공한 객체의 Key를 모아 두고, 처리 중 예외가 발생하면 해당 목록을 삭제하도록 구성했습니다.

아래는 업로드 함수의 예외 처리 부분입니다.

```java
} catch (BusinessException e) {
    deleteUploadedS3Keys(uploadedS3Keys);
    throw e;
} catch (RuntimeException e) {
    deleteUploadedS3Keys(uploadedS3Keys);
    throw e;
} catch (IOException e) {
    deleteUploadedS3Keys(uploadedS3Keys);
    throw new BusinessException(ErrorCode.ZIP_INVALID);
} finally {
    deleteTempDirectory(tempRoot);
}
```

[원본 코드 — FileServiceImpl.java, 업로드 예외 처리](https://github.com/H-SeungWoo/Asset-Box/blob/2bea504e71647dd744062e038e4fa01469efc91a/src/main/java/io/teabag/assetbox/file/service/FileServiceImpl.java#L429-L441)

보상 삭제는 객체별로 수행하며, 한 객체의 삭제가 실패해도 로그를 남기고 나머지 객체의 삭제를 계속 시도합니다.

```java
private void deleteUploadedS3Keys(List<String> uploadedS3Keys) {
    for (String s3Key : uploadedS3Keys) {
        try {
            s3FileStorageService.delete(s3Key);
        } catch (Exception deleteException) {
            log.warn("Failed to compensate uploaded S3 object. s3Key = {}", s3Key, deleteException);
        }
    }
}
```

[원본 코드 — FileServiceImpl.java, 보상 삭제](https://github.com/H-SeungWoo/Asset-Box/blob/2bea504e71647dd744062e038e4fa01469efc91a/src/main/java/io/teabag/assetbox/file/service/FileServiceImpl.java#L533-L541)

## 3. 삭제 요청과 실제 파일 제거를 분리

게시글에 연결된 파일을 삭제할 때는 메타데이터에 삭제 상태와 **7일 뒤의 삭제 예정 시각**을 기록합니다. 매시 정각 배치는 유예 기간이 지났고 아직 S3 삭제가 완료되지 않은 파일을 조회합니다.

```text
삭제 요청 → 삭제 상태·예정 시각 기록 → 7일 유예
  → 정각 배치에서 S3 삭제 시도
     ├─ 성공: 삭제 완료 시각 기록
     └─ 실패: 미완료 상태 유지 → 다음 배치에서 재시도
```

S3 삭제가 성공한 뒤에만 완료 시각을 기록하므로, 삭제에 실패한 파일은 다음 배치의 조회 대상에 남습니다.

```java
for (File file : files) {
    try {
        s3FileStorageService.delete(file.getS3Key());
        file.markStorageDeleted();
    } catch (Exception e) {
        log.warn("Failed to purge storage object. fileId = {}, s3Key = {}", file.getId(), file.getS3Key(), e);
    }
}
```

[원본 코드 — StoragePurgeService.java, 대상 조회·삭제](https://github.com/H-SeungWoo/Asset-Box/blob/2bea504e71647dd744062e038e4fa01469efc91a/src/main/java/io/teabag/assetbox/file/service/StoragePurgeService.java#L25-L44) · [삭제 예정 시각 기록 — File.java](./src/main/java/io/teabag/assetbox/file/domain/File.java)

## 구현과 함께 볼 테스트

| 확인할 동작 | 테스트 파일 |
| --- | --- |
| 경로 이탈·중복 경로·용량·파일 수 제한, 모델 분류 | [ZipExtractServiceTest.java](./src/test/java/io/teabag/assetbox/file/service/ZipExtractServiceTest.java) |
| ZIP·모델·텍스처 저장, 업로드 실패 시 보상 삭제, 삭제 유예 | [FileServiceTest.java](./src/test/java/io/teabag/assetbox/file/service/FileServiceTest.java) |
| S3 삭제 성공·실패에 따른 완료 처리 | [StoragePurgeServiceTest.java](./src/test/java/io/teabag/assetbox/file/service/StoragePurgeServiceTest.java) |
| 게시글과 파일의 업로드·조회·삭제 흐름 | [AssetPostFileLifecycleIntegrationTest.java](./src/test/java/io/teabag/assetbox/post/integration/AssetPostFileLifecycleIntegrationTest.java) |

## 기술 및 실행 안내

현재 저장소는 **Java 25 / Spring Boot 4.0.6 / Spring Data JPA / MySQL / AWS S3**를 사용합니다. 서비스 전체에는 Redis, Spring Security·OAuth2·JWT가 사용되며 Docker Compose와 GitHub Actions 기반 배포 구성이 포함되어 있습니다.

JDK 25와 실행 환경에 맞는 DB·Redis·S3·OAuth 설정이 필요합니다. 환경 설정과 컨테이너 배포 절차는 [DEPLOY.md](./DEPLOY.md), 운영 환경 변수 예시는 [.env.production.example](./.env.production.example)을 참고하세요.

```bash
# 테스트
./gradlew test

# 환경 설정 후 백엔드 실행
./gradlew bootRun
```
