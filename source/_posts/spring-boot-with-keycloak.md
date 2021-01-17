---
title: 使用Keycloak保護Spring Boot REST服務
date: 2021-01-09 16:33:29
tags:
  - spring
  - keycloak
toc: true
---

對於任何應用程序，安全性始終是主要的跨領域關注點。傳統上，我們過去在整體後端服務中只有一個安全層來保護所有前端應用程序請求。隨著新興的微服務架構的發展，不再有單一的後端服務。前端應用程序調用多個後端服務，這些服務又可以調用其他服務。不再有可以處理身份驗證的單層。由於微服務都具有許多較小的服務，每個服務都處理一個不同的職責，因此安全性的明顯解決方案是身份驗證和授權服務。這是Keycloak進行救援的地方，因為它提供了保護微服務所需的功能。

<!-- more -->

# 什麼是Keycloak
Keycloak是Red Hat贊助的開源身份和訪問管理（IAM）解決方案。它使我們能夠保護各種現代的前端應用程序和後端服務。Keycloak可以毫不費力地為我們的應用程序和服務添加身份驗證和授權。我們不需要處理用戶的存儲或身份驗證，因為所有這些都可以在Keycloak中直接使用。

Keycloak提供的功能包括單點登錄（SSO），身份代理和社交登錄，用戶聯合，客戶端適配器，管理員和用戶帳戶管理控制台。

![](https://i.imgur.com/0lR0sVR.png)

除了這些基本功能，Keycloak在其他IAM解決方案中脫穎而出的原因還在於其可定制的網頁和電子郵件模板主題，以及可擴展的功能和領域。可定制的主題很重要，因為它們使開發人員可以定制面向最終用戶的網頁的外觀，以便可以將其與應用程序緊密集成。Keycloak還提供了擴展其核心功能和範圍的功能。這包括可能：

+ 將自定義REST端點添加到Keycloak服務器
+ 添加 Server Provider Interface(SPI)
+ 將自定義 Java Persistence API(JPA)實體添加到Keycloak數據模型中

# 設置Keycloak
以下安裝並運行Keycloak服務器
1. 先確保要運行的機器有OpenJDK1.8或更高的版本
2. [Keycloak官方網站](https://www.keycloak.org/downloads.html)下載最新的Keycloak realease版本

**windows**
```
bin\standalone.bat
```

**linux**
```
bin/standalone.sh
```
Keycloak默認使用**H2資料庫**儲存數據，位於`standalone/data`目錄中。

此資料庫並不適用於Production環境，Keycloak官方強烈建議我們將其替換為更利於Productio環境的外部數據庫，例如PostgreSQL。

可以從[官方文檔](https://www.keycloak.org/docs/latest/server_installation/#_database)中獲取有關設置Keycloak數據庫的更多詳細信息。

## 建立初始化Admin User
Keycloak不附帶**初始admin**用戶，這意味著在開始使用Keycloak之前，您需要創建一個admin用戶。

打開[http://localhost:8080/auth](http://localhost:8080/auth)並填寫創建初始管理表單，用戶名為admin，密碼為password。

![](https://i.imgur.com/3GNGILZ.png)

單擊創建並進入**admin console**，然後使用剛剛創建的admin登錄。

![](https://i.imgur.com/c5dRXaZ.png)

登錄後Keycloak顯示的第一個屏幕是**Master Realm**詳細信息。

![](https://i.imgur.com/7nZSwLR.png)

+ Keycloak中的**realm**就像一個容器，其中包含並隔離user、role、group和client的集合。
+ Keycloak帶有一個預先創建的Master realm。
+ 建議不要使用Master realm管理組織中user和application。
+ 創建和管理其他realm，為admin保留對Master realm的使用。

![](https://i.imgur.com/5iS9L10.png)

## 建立Realms
我們將需要一個realm來管理我們的Spring Boot REST服務使用的user，role和client。

因此，讓我們建立第一個realm。

將鼠標懸停在左上角顯示"Master"的下拉菜單上，然後單擊"Add realm"按鈕。

![](https://i.imgur.com/sFDqA9O.png)

我們將第一個realm命名為demo：

![](https://i.imgur.com/LSYaOT8.png)

點擊create，將會被定向到創建的realm詳細信息頁面，可以在這進一步配置realm。

![](https://i.imgur.com/WhYm21L.png)

## 建立Roles
role對user進行分類。在應用程序中，**訪問資源的權限通常授予role而不是user**。

Admin、User和Manager都是組織中可能存在的典型角色。

要建立role，請單擊左側的"Roles"，然後點擊頁面上的"Add Role"按鈕。

![](https://i.imgur.com/7MM55ib.png)

我們將role命名為spring-user，然後點擊save。

![](https://i.imgur.com/sH6EtKT.png)

## 建立Users
### 建立User基本資料
單擊左側的"Users"，然後點擊"Add User"按鈕。

![](https://i.imgur.com/hSbshmr.png)

將username填寫為Steven`(登入的account)`，將First Name填寫為Steven，並保留其他所有默認設置。

確保"User Enabled"為"on"。點擊save建立第一個user

![](https://i.imgur.com/MNP17PY.png)

### 為User設置密碼
Steven需要密碼才能登錄Keycloak，所以需要為Steven創建一個密碼。

點擊"Credentials"標籤，然後在密碼和確認密碼輸入*mypassword*。

且設置**Temporary**為**OFF**，以便在下次登錄時無需更改密碼。

點擊"Set Password"以保存Steven的密碼憑證。

![](https://i.imgur.com/fg1mWxz.png)

### 分配Role給User
將之前創建的role: `spring-user`分配給Steven。

為此，請點擊"Role Mappings"選項，選擇spring-user角色，然後點擊"Add Selected"。

![](https://i.imgur.com/TCFK9gy.png)

## 建立Clients
### Client基本資料建立
> Clients are entities that will request the authentication of a user.

Client有兩種形式。
1. 第一種類型的Client是想要參與SSO的應用程式，這些Client只希望Keycloak為他們提供安全性。
2. 另一種Client是請求訪問令牌的Client，以便它可以代表已認證的用戶調用其他服務。

我們將創建一個Client用於保護我們的Spring Boot REST服務。

點擊左側的"**Clients**"，然後點選"**Create**"。

![](https://i.imgur.com/E66DlEQ.png)

在表單中，將Client Id填寫為**spring-boot**，為Client Protocol選擇**OpenID Connect**，接著點選Save。

![](https://i.imgur.com/HtT11cE.png)

### 更改Access Type及Authorization Enabled
在Client的Settings頁面中，我們需要將**Access Type**更改為`confidential`，而不是默認的`public`(不需要Client Secret)。存檔之前，請打開"**Authorization Enabled**"開關。

![](https://i.imgur.com/UGSiFEc.png)

### Client Secret
在Client的Credentials頁面中，先將Secret內容複製出來，下面內容會使用到

![](https://i.imgur.com/W7IuZY1.png)


## 拿取Access Token
Client通過使用HTTP POST方法向Keycloak中的Token endpoint發出**Token交換請求**來請求安全令牌。
```
/auth/realms/{realm}/protocol/openid-connect/token
```

- Keycloak中的Token交換請求是[IETF](https://tools.ietf.org/html/rfc8693)上**OAuth 2.0Token交換**規範的寬鬆實現。

- **OpenID Connect** Token endpoint 上的簡單授予類型調用。

- Keycloak使用"**application/x-www-form-urlencoded**"格式和UTF-8字符編碼來接受HTTP請求實體中的參數。

打開Postman，建立對`http://localhost:8080/auth/realms/demo/protocol/openid-connect/token` POST請求

![](https://i.imgur.com/NDXsvJF.png)

會收到帶有**Access Token**和**Refresh Token**以及其他附帶詳細信息的JSON Response。

![](https://i.imgur.com/vFO75lm.png)

將接收到的Access Token作為Bearer Token放置在Authorization Header中，就可以在受到Keycloak安全保護的REST API進行請求:
```
headers: {
     'Authorization': 'Bearer ' + {access_token}
 }
```

從Keycloak的**Admin console**中，進到spring-boot的Client詳細資訊，然後點選"**Sessions**"選項。您將看到Steven的登錄session。

![](https://i.imgur.com/wWeya3q.png)

點擊Steve進到User詳細資訊，然後點選"Sessions"選項。在"Sessions"選項卡下，有一些選項可用於**註銷**特定會話或**註銷所有會話**。

![](https://i.imgur.com/n5fT74P.png)

我們已經配置了Keycloak並能夠通過Postman請求Access Token，下一步是創建**Spring Boot REST服務**並使用Keycloak保護它。

# 建立一個Spring Boot應用程序

使用[Spring Initializr](https://start.spring.io/)網站生成具有Spring Boot 2.x依賴項的項目。還需要Spring Boot Starter Web 模塊

![](https://i.imgur.com/jYHeijY.png)

接下來，創建一個@RestController MessagingRestController Class，該Class將getMessage()方法公開為/user/message上的HTTP GET請求。此方法將返回"Hello, User"字串。
```java=
@RestController
public class MessagingRestController {

  @GetMapping(path = "/user/message")
  public String getUserMessage() {
    return "Hello, User";
  }
}
```

application.yaml中配置server.port
```yaml=
server:
  port: 8000
```

現在我們的REST服務已經可以運行了，我們需要使用Keycloak保護它。

# 使用Keycloak保護Spring Boot REST服務
為了保護您的Spring Boot REST服務，必須將Spring Boot的**Keycloak Adpater**添加到服務中。

build.gradle
```groovy=
ext {
	set('keycloakVersion', '12.0.1')
}

dependencyManagement {
	imports {
		mavenBom "org.keycloak.bom:keycloak-adapter-bom:${keycloakVersion}"
	}
}

dependencies {
	implementation 'org.keycloak:keycloak-spring-boot-starter'
}
```

接著將keycloak相關內容配置到application.yml
```yaml=
keycloak:
  auth-server-url: http://localhost:8080/auth
  realm: demo
  resource: spring-boot
  credentials:
    secret: 86ef845e-735a-42aa-84c7-fac294c359ad
  bearer-only: true
```

- auth-server-url: Keycloak服務器的URL
- realm: 在keycloak所建立的realm
- resource: 在keycloak所建立的Client Id
- secret: 在keycloak所建立的Client Secret
- only-bearer-only: 必須為true，以便Adpater不會嘗試對用戶進行身份驗證，而僅驗證Access Token

再讓我們為應用程式增加security-constraints`(安全性約束)`。此配置非常重要，因為Keycloak Adapter將根據我們的配置允許或拒絕對我們資源的訪問請求

下列例子為: 確保對URL:/user/*的每個請求，僅在該請求的用戶是具有spring-user角色的已認證用戶時才被授權使用。
```yaml=
keycloak:
  security-constraints:
    - auth-roles:
      - spring-user
      security-collections:
      - name: 
        patterns:
        - /user/*
```

在運行服務之前，讓我們打開Keycloak的DEBUG日誌記錄級別，以在console中查看更多詳細信息。
```yaml=
logging:
  level:
    org.keycloak: TRACE
```

## Test HTTP GET User Message

### Role spring-user
打開postman，URL輸入`http://localhost:8000/user/message`，HTTP Method為GET
再點擊"Authorization"標籤，然後選擇Bearer Token，將之前如下圖示中response的access_token貼上Token欄位裡。

1. 使用者Steven驗證

![](https://i.imgur.com/LcXn2sM.png)


2. 將上圖的access_token，貼入下圖Token欄位中

![](https://i.imgur.com/WlKirk0.png)

3. 點選"Send"過後，會收到 "Hello, User" response

![](https://i.imgur.com/KBPLQkc.png)

在Console中，會看到keycloak日誌紀錄

![](https://i.imgur.com/azfW1CZ.png)

### Role spring-admin
我們將使用新role和user對Keycloak進行更多配置，以演示我們為其他請求URL定義安全約束。

回到keycloak admin console，建立一個新的Role: `spring-admin`

![](https://i.imgur.com/C5RhXU5.png)

建立新的User: **Dave**並設置密碼，且授予**spring-admin**角色

![](https://i.imgur.com/0Zh2NNo.png)

![](https://i.imgur.com/LVoVTJu.png)

回到spring應用程式中，建立一個getAdminMessage endpoint
```java=
@GetMapping(path = "/admin/message")
public String getAdminMessage() {
  return "Hello, Admin";
}
```

最後，向我們的應用程序添加另一個安全約束，以授權具有spring-admin角色的用戶訪問URL /admin/*請求。

```yaml=
keycloak:
  security-constraints:
    - auth-roles:
        - spring-user
      security-collections:
        - name:
          patterns:
            - /user/*
    - auth-roles:
        - spring-admin
      security-collections:
        - name:
          patterns:
            - /user/*
            - /admin/*
```

現在我們使用Steven的token對/admin/message進行請求

此請求被拒絕，因為我們尚未授予spring-user角色訪問請求/admin/*的權限。
![](https://i.imgur.com/NSQjvjO.png)

讓我們換回用Dave的token，來對/admin/message進行請求，將能成功進行請求
![](https://i.imgur.com/6FKkNbw.png)

![](https://i.imgur.com/H1PDuxz.png)

# 結論
- 了解到Keycloak是一個現代的身份和訪問管理系統，提供了許多現成的功能。
- 還學習瞭如何使用realm、role、User和Client來設置Keycloak。
- 最後，我們學習如何配置Spring Boot REST服務以利用Keycloak認證和授權所有請求。

# 參考資料
- https://codeburst.io/securing-spring-boot-rest-services-using-keycloak-without-writing-code-7c47ab72fb9d