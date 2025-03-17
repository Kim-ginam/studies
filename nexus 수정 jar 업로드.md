### 🔍 **수정된 파일만 Nexus에 업로드하는 방법**
지금은 **모든 `.jar` 파일을 반복문으로 업로드하고 있지만**,  
**나중에는 변경된 파일만 업로드하고 싶다면, 파일의 변경 여부를 체크하는 방법이 필요**합니다.

---

## ✅ **1. 방법 1: `Get-Item`을 사용하여 최근 수정된 파일만 업로드**
Windows에서는 **PowerShell의 `LastWriteTime` 속성을 이용**하여  
최근 수정된 `.jar` 파일만 Nexus에 업로드할 수 있습니다.

📌 **PowerShell 스크립트 예제 (최근 24시간 이내 수정된 파일만 업로드)**  
```powershell
# Nexus 정보 설정
$NEXUS_URL = "http://<NEXUS_HOST>:8081/repository/maven-releases/"
$M2Repo = "$env:USERPROFILE\.m2\repository"

# 최근 24시간 이내 수정된 모든 JAR 파일 찾기
$JarFiles = Get-ChildItem -Path $M2Repo -Recurse -Filter "*.jar" | Where-Object { $_.LastWriteTime -gt (Get-Date).AddHours(-24) }

foreach ($JarFile in $JarFiles) {
    $AbsoluteFilePath = "`"$($JarFile.FullName)`""
    $PomFile = "`"$($AbsoluteFilePath -replace "\.jar$", ".pom")`""

    # groupId, artifactId, version 추출
    $RelativePath = $JarFile.FullName -replace [regex]::Escape($M2Repo + "\"), ""
    $PathParts = $RelativePath -split "\\"
    $Version = $PathParts[-2]
    $ArtifactId = $PathParts[-3]
    $GroupId = "`"$(($PathParts[0..($PathParts.Length - 3)] -join "."))`""

    Write-Host "Uploading: $AbsoluteFilePath (Group: $GroupId, Artifact: $ArtifactId, Version: $Version)"

    # POM 파일이 존재하는 경우
    if (Test-Path $PomFile) {
        mvn deploy:deploy-file `
          -DgroupId=$GroupId `
          -DartifactId=$ArtifactId `
          -Dversion=$Version `
          -Dpackaging="jar" `
          -Dfile=$AbsoluteFilePath `
          -DpomFile=$PomFile `
          -DrepositoryId="nexus" `
          -Durl=$NEXUS_URL
    } else {
        mvn deploy:deploy-file `
          -DgroupId=$GroupId `
          -DartifactId=$ArtifactId `
          -Dversion=$Version `
          -Dpackaging="jar" `
          -Dfile=$AbsoluteFilePath `
          -DrepositoryId="nexus" `
          -Durl=$NEXUS_URL `
          -DgeneratePom=true
    }
}
```

✅ **이제 최근 24시간 이내 변경된 `.jar` 파일만 Nexus에 업로드됨**  
⏰ `-gt (Get-Date).AddHours(-24)` 값 변경하면, 특정 기간 동안 수정된 파일만 업로드 가능  
   - `-24` → 최근 24시간 이내 변경된 파일  
   - `-48` → 최근 48시간 (2일) 이내 변경된 파일  

---

## ✅ **2. 방법 2: Git을 사용하여 변경된 파일만 업로드**
만약 **Git을 사용하고 있다면**, Git에서 변경된 `.jar` 파일만 업로드할 수도 있습니다.

📌 **PowerShell 스크립트 (Git 변경된 파일만 업로드)**
```powershell
# Nexus 정보 설정
$NEXUS_URL = "http://<NEXUS_HOST>:8081/repository/maven-releases/"
$M2Repo = "$env:USERPROFILE\.m2\repository"

# Git에서 변경된 파일 찾기
$ModifiedFiles = git diff --name-only HEAD | Where-Object { $_ -match "\.jar$" }

foreach ($File in $ModifiedFiles) {
    $AbsoluteFilePath = "`"$M2Repo\$File`""
    $PomFile = "`"$($AbsoluteFilePath -replace "\.jar$", ".pom")`""

    # groupId, artifactId, version 추출
    $RelativePath = $File -replace [regex]::Escape($M2Repo + "\"), ""
    $PathParts = $RelativePath -split "\\"
    $Version = $PathParts[-2]
    $ArtifactId = $PathParts[-3]
    $GroupId = "`"$(($PathParts[0..($PathParts.Length - 3)] -join "."))`""

    Write-Host "Uploading: $AbsoluteFilePath (Group: $GroupId, Artifact: $ArtifactId, Version: $Version)"

    # POM 파일이 존재하는 경우
    if (Test-Path $PomFile) {
        mvn deploy:deploy-file `
          -DgroupId=$GroupId `
          -DartifactId=$ArtifactId `
          -Dversion=$Version `
          -Dpackaging="jar" `
          -Dfile=$AbsoluteFilePath `
          -DpomFile=$PomFile `
          -DrepositoryId="nexus" `
          -Durl=$NEXUS_URL
    } else {
        mvn deploy:deploy-file `
          -DgroupId=$GroupId `
          -DartifactId=$ArtifactId `
          -Dversion=$Version `
          -Dpackaging="jar" `
          -Dfile=$AbsoluteFilePath `
          -DrepositoryId="nexus" `
          -Durl=$NEXUS_URL `
          -DgeneratePom=true
    }
}
```

✅ **이제 Git에서 변경된 `.jar` 파일만 Nexus에 업로드됨**  
📌 **Git이 없는 환경이면 사용 불가 → 방법 1 사용 권장**  

---

## ✅ **3. 방법 3: Nexus API를 사용하여 파일 비교 후 업로드**
Nexus API를 이용하면 **Nexus에 이미 존재하는 파일과 비교한 후 업로드 가능**  
이 방식은 **네트워크 트래픽을 줄일 수 있는 장점이 있음**

📌 **PowerShell 스크립트 (Nexus API로 존재 여부 확인 후 업로드)**
```powershell
# Nexus 정보 설정
$NEXUS_URL = "http://<NEXUS_HOST>:8081/repository/maven-releases/"
$NEXUS_USER = "admin"
$NEXUS_PASS = "admin123"

# .m2 저장소 경로
$M2Repo = "$env:USERPROFILE\.m2\repository"

# 모든 JAR 파일 찾기
$JarFiles = Get-ChildItem -Path $M2Repo -Recurse -Filter "*.jar"

foreach ($JarFile in $JarFiles) {
    $AbsoluteFilePath = "`"$($JarFile.FullName)`""
    $ArtifactPath = $JarFile.FullName -replace [regex]::Escape($M2Repo + "\"), ""

    # Nexus에 파일 존재 여부 확인
    $NexusCheckUrl = "$NEXUS_URL$ArtifactPath"
    $Response = Invoke-WebRequest -Uri $NexusCheckUrl -Method Head -UseBasicParsing -Credential (New-Object PSCredential $NEXUS_USER, (ConvertTo-SecureString $NEXUS_PASS -AsPlainText -Force))

    if ($Response.StatusCode -eq 404) {
        Write-Host "Uploading: $AbsoluteFilePath"
        mvn deploy:deploy-file `
          -DgroupId="com.example" `
          -DartifactId="my-library" `
          -Dversion="1.0.0" `
          -Dpackaging="jar" `
          -Dfile=$AbsoluteFilePath `
          -DrepositoryId="nexus" `
          -Durl=$NEXUS_URL
    } else {
        Write-Host "File already exists in Nexus, skipping upload: $AbsoluteFilePath"
    }
}
```

✅ **Nexus에서 존재하지 않는 파일만 업로드하여 불필요한 중복 업로드 방지 가능**  

---

## 🚀 **결론 (추천 방법)**
1️⃣ **최근 수정된 파일만 업로드** (`LastWriteTime` 사용) → 💡 **가장 간단하고 효과적**  
2️⃣ **Git에서 변경된 파일만 업로드** (`git diff` 사용) → 💡 **Git을 사용한다면 추천**  
3️⃣ **Nexus API를 이용하여 기존 파일과 비교 후 업로드** → 💡 **네트워크 최적화 필요할 때 추천**  

✅ **이제 필요할 때만 수정된 `.jar` 파일을 Nexus에 업로드할 수 있습니다! 🚀**  
🚀 **테스트 후 결과를 알려주세요! 😊**