윈도우 환경에서 `.m2/repository`에 있는 라이브러리들을 다른 서버의 Nexus에 업로드하는 방법을 정리해보겠습니다.

---

## ✅ 방법 1: PowerShell 스크립트를 사용한 자동 업로드 (추천)
`.m2/repository/` 내 모든 `.jar` 파일을 Nexus에 자동 업로드하는 **PowerShell 스크립트**를 사용합니다.

### **1. 사전 준비**
- Windows에서 `mvn deploy:deploy-file`이 실행 가능해야 합니다.
- Nexus에 접속 가능한 URL 확인 (`http://<NEXUS_HOST>:8081/repository/maven-releases/`)
- 관리자 계정 정보 (`admin/admin123`)

---

### **2. PowerShell 스크립트 작성**
아래 스크립트를 `upload_to_nexus.ps1`로 저장하세요.

```powershell
# Nexus 정보 설정
$NexusUrl = "http://<NEXUS_HOST>:8081/repository/maven-releases/"
$NexusUser = "admin"
$NexusPass = "admin123"

# .m2 저장소 경로 (기본값)
$M2Repo = "$env:USERPROFILE\.m2\repository"

# 모든 JAR 파일 찾기
$JarFiles = Get-ChildItem -Path $M2Repo -Recurse -Filter "*.jar"

foreach ($JarFile in $JarFiles) {
    $PomFile = $JarFile.FullName -replace "\.jar$", ".pom"

    # groupId, artifactId, version 추출
    $RelativePath = $JarFile.FullName -replace [regex]::Escape($M2Repo + "\"), ""
    $GroupId = ($RelativePath | Split-Path -Parent) -replace "\\", "."
    $ArtifactId = $JarFile.BaseName
    $Version = Split-Path -Path $RelativePath -Parent | Split-Path -Leaf

    Write-Host "Uploading: $JarFile (Group: $GroupId, Artifact: $ArtifactId, Version: $Version)"

    # Maven 명령어 실행
    mvn deploy:deploy-file `
      -DgroupId=$GroupId `
      -DartifactId=$ArtifactId `
      -Dversion=$Version `
      -Dpackaging=jar `
      -Dfile="$JarFile" `
      -DpomFile="$PomFile" `
      -DrepositoryId=nexus `
      -Durl=$NexusUrl `
      -DgeneratePom=true
}
```

---

### **3. PowerShell 스크립트 실행**
1. PowerShell을 **관리자 권한**으로 실행
2. `upload_to_nexus.ps1`이 있는 폴더로 이동
3. 아래 명령어 실행:
   ```powershell
   Set-ExecutionPolicy Unrestricted -Scope Process
   ./upload_to_nexus.ps1
   ```

🔹 **장점**:
- `.m2/repository/` 내 모든 라이브러리를 자동 업로드
- `groupId`, `artifactId`, `version`을 자동으로 추출하여 Nexus에 업로드
- Nexus 서버가 원격에 있어도 업로드 가능

🔹 **단점**:
- Nexus와 연결이 원활해야 함 (방화벽 문제 확인 필요)
- `mvn deploy:deploy-file`을 여러 번 실행해야 하므로 시간이 걸릴 수 있음

---

## ✅ 방법 2: `.m2/repository` 전체 압축 후 원격 업로드
파일 개수가 많다면, `.m2/repository/`를 압축해서 원격 서버로 보내고 Nexus에서 직접 배포하는 방법도 가능합니다.

### **1. `.m2/repository`를 압축**
```powershell
Compress-Archive -Path "$env:USERPROFILE\.m2\repository\*" -DestinationPath ".\maven-repo.zip"
```

### **2. 압축 파일을 원격 서버로 전송 (SCP/FTP)**
FTP를 사용할 경우:
```powershell
$ftpServer = "ftp://<NEXUS_SERVER_IP>/uploads/"
$ftpUser = "admin"
$ftpPass = "admin123"
$sourceFile = ".\maven-repo.zip"

$webClient = New-Object System.Net.WebClient
$webClient.Credentials = New-Object System.Net.NetworkCredential($ftpUser, $ftpPass)
$webClient.UploadFile("$ftpServer/maven-repo.zip", $sourceFile)
```

### **3. 원격 서버에서 압축 해제**
서버에서 압축을 풀고, 해당 파일들을 수동으로 업로드하면 됩니다.
```sh
unzip maven-repo.zip -d /opt/nexus-data/maven-releases/
```

---

## ✅ 결론 (추천 방법)
### **1. 개별 업로드 → PowerShell 스크립트 사용 (`upload_to_nexus.ps1`)**
- **장점:** Nexus 서버와 직접 연결되어 있어 바로 업로드 가능
- **단점:** 실행 시간이 오래 걸릴 수 있음

### **2. `.m2/repository` 전체 압축 후 업로드**
- **장점:** Nexus 서버에서 직접 처리 가능
- **단점:** 서버에서 추가 작업 필요

🚀 **PowerShell 스크립트 방식**이 Nexus 서버와 연결이 가능하다면 가장 효율적인 방법입니다!