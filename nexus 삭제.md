### 🔍 **현재 문제 분석 (Nexus에 있는 라이브러리를 한 번에 삭제하는 방법)**
✅ **Nexus에 잘못 업로드된 라이브러리(Artifact)들을 한 번에 삭제하고 싶음**  
✅ **Nexus 웹 UI에서는 개별적으로 삭제해야 하므로 번거로움**  
✅ **스크립트나 API를 사용하여 한 번에 삭제할 방법 필요**  

📌 **Nexus에는 REST API를 제공하므로, 이를 활용하여 대량 삭제 가능**  
📌 **아래 방법을 사용하여 한 번에 삭제 가능**  

---

## ✅ **1. Nexus REST API를 이용한 일괄 삭제**
📌 **Nexus 3의 REST API를 활용하여 특정 그룹의 라이브러리를 한 번에 삭제할 수 있음**  

### **📌 1) 삭제할 라이브러리 확인**
먼저, 삭제할 라이브러리 목록을 확인하려면 아래 API를 호출하면 됨:
```powershell
curl -u admin:iteyes7979 -X GET "http://192.168.121.183:8081/service/rest/v1/components?repository=maven-releases"
```
✅ **출력된 JSON에서 삭제할 라이브러리의 `id` 값을 확인**  

---

### **📌 2) 특정 라이브러리 삭제 (개별 삭제)**
📌 **위에서 확인한 `id` 값을 사용하여 개별 삭제 가능**
```powershell
curl -u admin:iteyes7979 -X DELETE "http://192.168.121.183:8081/service/rest/v1/components/{id}"
```
✅ **이 명령어를 여러 개 실행하면 여러 개의 라이브러리를 삭제 가능**  

---

### **📌 3) 특정 그룹(`groupId`)에 해당하는 라이브러리 일괄 삭제**
📌 **PowerShell 스크립트로 특정 `groupId`에 해당하는 라이브러리를 한 번에 삭제**
```powershell
$NexusUrl = "http://192.168.121.183:8081"
$Repo = "maven-releases"
$User = "admin"
$Pass = "iteyes7979"

# 특정 그룹 ID 설정
$GroupId = "org.springframework.cloud"

# 해당 그룹의 모든 라이브러리 조회
$Artifacts = Invoke-RestMethod -Uri "$NexusUrl/service/rest/v1/components?repository=$Repo" -Method Get -Credential (New-Object System.Management.Automation.PSCredential ($User, (ConvertTo-SecureString $Pass -AsPlainText -Force)))

foreach ($Artifact in $Artifacts.items) {
    if ($Artifact.group -eq $GroupId) {
        Write-Host "Deleting: $($Artifact.id)"
        Invoke-RestMethod -Uri "$NexusUrl/service/rest/v1/components/$($Artifact.id)" -Method Delete -Credential (New-Object System.Management.Automation.PSCredential ($User, (ConvertTo-SecureString $Pass -AsPlainText -Force)))
    }
}
```
✅ **이렇게 하면 특정 `groupId`에 해당하는 모든 라이브러리를 한 번에 삭제 가능!** 🚀  

---

### **📌 4) 특정 저장소(`maven-releases`)의 모든 라이브러리 삭제**
📌 **Nexus 저장소 전체를 비우고 싶다면 아래 스크립트를 실행**
```powershell
$NexusUrl = "http://192.168.121.183:8081"
$Repo = "maven-releases"
$User = "admin"
$Pass = "iteyes7979"

# 모든 라이브러리 조회
$Artifacts = Invoke-RestMethod -Uri "$NexusUrl/service/rest/v1/components?repository=$Repo" -Method Get -Credential (New-Object System.Management.Automation.PSCredential ($User, (ConvertTo-SecureString $Pass -AsPlainText -Force)))

foreach ($Artifact in $Artifacts.items) {
    Write-Host "Deleting: $($Artifact.id)"
    Invoke-RestMethod -Uri "$NexusUrl/service/rest/v1/components/$($Artifact.id)" -Method Delete -Credential (New-Object System.Management.Automation.PSCredential ($User, (ConvertTo-SecureString $Pass -AsPlainText -Force)))
}
```
✅ **이렇게 하면 `maven-releases` 저장소에 있는 모든 라이브러리를 삭제할 수 있음** 🚀  

---

## 🚀 **결론 (Nexus에서 한 번에 라이브러리 삭제)**
1️⃣ **REST API를 사용하여 삭제할 라이브러리 목록 확인 (`GET /components?repository=maven-releases`)**  
2️⃣ **특정 라이브러리 개별 삭제 (`DELETE /components/{id}`)**  
3️⃣ **특정 `groupId`에 해당하는 라이브러리 일괄 삭제 (PowerShell 스크립트 사용)**  
4️⃣ **저장소 전체를 비우고 싶다면 `maven-releases` 내 모든 라이브러리 삭제 스크립트 실행**  

✅ **이제 한 번에 라이브러리를 삭제할 수 있음! 🚀**  
🚀 **실행 후 결과를 알려주세요! 😊**