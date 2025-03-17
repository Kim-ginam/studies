### 🔍 **Nexus만 바라보도록 설정하여 오프라인 환경에서 빠르게 빌드하는 방법**
현재 `.jar` 파일들을 Nexus에 올렸기 때문에 **Maven이 Nexus만 바라보게 설정하면 됩니다.**  
✅ **다른 Repository(Maven Central 포함)에 접근하지 않도록 설정하면, 불필요한 네트워크 연결 시도 없이 빠르게 빌드 가능** 🚀

---

## ✅ **1. `settings.xml` 수정하여 Maven이 Nexus만 바라보도록 설정**
📌 **모든 Maven 프로젝트에서 Nexus 외의 Repository를 사용하지 않도록 설정**

**📌 `C:\Users\user\.m2\settings.xml` (Windows) 또는 `~/.m2/settings.xml` (Linux, macOS)**
```xml
<mirrors>
    <!-- Maven이 Nexus만 사용하도록 설정 -->
    <mirror>
        <id>nexus</id>
        <mirrorOf>*</mirrorOf>
        <url>http://<NEXUS_HOST>:8081/repository/maven-releases/</url>
        <layout>default</layout>
    </mirror>
</mirrors>

<profiles>
    <profile>
        <id>nexus</id>
        <repositories>
            <repository>
                <id>nexus</id>
                <url>http://<NEXUS_HOST>:8081/repository/maven-releases/</url>
                <releases>
                    <enabled>true</enabled>
                </releases>
                <snapshots>
                    <enabled>false</enabled>
                </snapshots>
            </repository>
        </repositories>

        <pluginRepositories>
            <pluginRepository>
                <id>nexus-plugins</id>
                <url>http://<NEXUS_HOST>:8081/repository/maven-releases/</url>
                <releases>
                    <enabled>true</enabled>
                </releases>
                <snapshots>
                    <enabled>false</enabled>
                </snapshots>
            </pluginRepository>
        </pluginRepositories>
    </profile>
</profiles>

<activeProfiles>
    <activeProfile>nexus</activeProfile>
</activeProfiles>
```

✅ **이제 Maven은 오직 Nexus에서만 라이브러리를 가져오며, Maven Central이나 다른 Repository를 조회하지 않음.**  
✅ **오프라인 환경에서도 네트워크 타임아웃 없이 빠르게 동작!** 🚀  

---

## ✅ **2. `pom.xml`에서 다른 Repository 제거 (선택 사항)**
📌 **각 프로젝트의 `pom.xml`에서 `repositories` 항목을 삭제하여 Maven이 Nexus만 바라보게 할 수도 있음.**

```xml
<repositories>
    <repository>
        <id>nexus</id>
        <url>http://<NEXUS_HOST>:8081/repository/maven-releases/</url>
        <releases>
            <enabled>true</enabled>
        </releases>
        <snapshots>
            <enabled>false</enabled>
        </snapshots>
    </repository>
</repositories>
```

✅ **이제 `pom.xml`에서도 Nexus 외에는 참조하지 않음**  
✅ **불필요한 네트워크 연결 시도를 방지하여 빌드 속도 향상**  

---

## ✅ **3. Maven 빌드를 강제로 오프라인 모드에서 실행하여 테스트**
📌 **Maven이 Nexus만 바라보는지 확인하기 위해 오프라인 모드(`-o` 옵션)로 빌드 테스트**

```powershell
mvn clean install -o
```

✅ **오프라인 모드(`-o`)에서 빌드가 정상적으로 완료되면, Nexus 외에는 어떤 Repository도 사용하지 않는 것!**  

---

## ✅ **4. Maven이 여전히 외부 Repository를 조회하는 경우 확인해야 할 것**
📌 **Maven이 Nexus 외부의 Repository를 조회하는 로그가 남는다면, 다음을 확인해야 함.**

### **🔹 1) `pom.xml`에서 `repositories` 설정이 있는지 확인**
✅ `repositories` 섹션에 Maven Central(`https://repo.maven.apache.org/maven2`)이나 다른 Repository가 설정되어 있다면 삭제  

```xml
<repositories>
    <repository>
        <id>nexus</id>
        <url>http://<NEXUS_HOST>:8081/repository/maven-releases/</url>
    </repository>
</repositories>
```

---

### **🔹 2) `settings.xml`에 Nexus 외의 Repository 설정이 남아있는지 확인**
✅ `settings.xml`의 `<mirrors>` 섹션에서 `mirrorOf="*"`이 제대로 설정되어 있어야 함.

```xml
<mirrors>
    <mirror>
        <id>nexus</id>
        <mirrorOf>*</mirrorOf>
        <url>http://<NEXUS_HOST>:8081/repository/maven-releases/</url>
    </mirror>
</mirrors>
```

✅ **이 설정이 없으면 Maven이 Maven Central을 계속 조회할 수 있음!** 🚨  

---

## 🚀 **결론 (완전한 오프라인 Nexus 설정)**
1️⃣ **Maven `settings.xml`에서 `mirrorOf="*"`로 Nexus만 바라보도록 설정**  
2️⃣ **각 프로젝트(`pom.xml`)에서 `repositories` 항목을 Nexus만 바라보도록 수정**  
3️⃣ **Maven 빌드를 `-o` (오프라인 모드)에서 실행하여 테스트 (`mvn clean install -o`)**  
4️⃣ **설정 후에도 Maven Central을 조회하는 로그가 남으면 `settings.xml`과 `pom.xml`에서 불필요한 Repository를 제거**  

✅ **이제 Maven은 오직 Nexus에서만 라이브러리를 가져오며, 오프라인 환경에서도 빠르게 동작할 것입니다! 🚀**  
🚀 **테스트 후 정상적으로 동작하는지 확인하고, 추가 질문이 있으면 알려주세요! 😊**