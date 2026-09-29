# CLAUDE.md

這是 **BizDock** 的總管 repo。BizDock 是 theAgileFactory 開發的開源 PPM（專案組合管理）Web 應用程式。這個 repo 本身沒有程式碼，最上層的每個目錄都是指向 `github.com/theAgileFactory/*` 的 **git submodule**（見 `.gitmodules`）。修改時，先在對應的 submodule 內 commit，必要時再回到這個 repo commit 更新後的 submodule 指標。

產品文件：https://help.bizdock.io

## 技術堆疊（舊版，約 2015–2019 年）

- Java 1.8（`java.source/target=1.8`）。原始碼編碼為 **latin1**，原始碼中請避免使用非 ASCII 字元。
- Play Framework **2.4.2**（Java）、Scala 2.11、SBT 0.13.8 / Typesafe Activator 1.3.7。
- ORM 用 Ebean（`com.avaje.ebean`），授權用 Deadbolt 2.4，SSO（CAS/SAML）用 pac4j，API 用 Swagger 1.3（wordnik），非同步 job 用 Akka，報表用 JasperReports。
- 資料庫為 MariaDB/MySQL（connector 5.1）。Schema 由 **MyBatis Migrations** 管理，不使用 Play evolutions。
- 建置：**Maven** 搭配 `play2-maven-plugin`（packaging 為 `play2`）。`build.sbt` / `project/` 只用於 Activator `run` 和 IDE 支援。
- CI：Travis（`.travis.yml`、`.travis/*.sh`）建置 `master`/`R17` branch，並把簽章後的 artifact 發佈到 Maven Central（`com.sword-group.bizdock.*`）。`*.gpg.enc` 和 `deploy_key.enc` 是加密的 CI secret，請勿更動。
- 這些 repo **沒有任何自動化測試**。

`app-framework/MIGRATION_GUIDE.md` 是新加入的升級計畫，目標是 Java 21 / Play 3 / Scala 2.13。內容描述的是目標狀態，**尚未**實作。

## Submodule 與 dependency 順序

目前主要元件的發佈版本：**17.3.1**。

| Submodule | Artifact | 角色 |
|---|---|---|
| `app-framework` | `com.sword-group.bizdock.lib:app-framework`（play2） | 技術基礎：security、各項 service（account、job、plugin、storage、email、KPI、custom attribute、i18n、API）、UI utility（`framework.utils.Table`、`Menu`、`Pagination` 等）、framework model（`models.framework_models`） |
| `maf-desktop-datamodel` | `...desktop:maf-desktop-datamodel`（play2） | Ebean entity，位於 `app/models/{pmo,finance,governance,delivery,timesheet,reporting,architecture,datasyndication,common}`，另有 `app/constants` |
| `maf-desktop-app` | `...desktop:maf-desktop-app`（play2） | 主應用程式：controller、DAO、service、view、`conf/routes`。`development/` 內也有**開發工具** |
| `maf-defaultplugins-extension` | `...lib:maf-defaultplugins-extension`（play2） | 預設整合 plugin（Atlassian/JIRA、Redmine、Jenkins、Nexus、Subversion、system）與 widget，打包成 extension（`conf/extension.xml`） |
| `dbmdl-framework` | `...dbmdl:dbmdl-framework`（pom/zip） | framework table 的 MyBatis migration script |
| `maf-dbmdl` | `...dbmdl:maf-dbmdl`（pom/zip） | 業務 table 的 MyBatis migration script |
| `bizdock-packaging` | `...packaging:maf-desktop` | 把 Play dist 和預設 extension 組裝成可部署的 package |
| `bizdock-installation` | `...packaging:bizdock-image-builder` | Dockerfile（`bizdock/`、以 MariaDB 為基底的 `bizdockdb/`）、`build_image.sh`，以及管理 instance 的 `cli/create.sh` |
| `bizdock-docker` | – | `development-bizdock-image/`：Docker 開發環境（`use/bizdockctl.sh`、`build.sh`、`db.sh`） |
| `replacer` | `com.agifac.deploy:replacer-maven-plugin` | Maven plugin，依環境替換打包後 zip 內的 `${prop}` placeholder |
| `jira-plugin-api` | `com.agifac.lib:jira-plugin-api`（atlassian-plugin，`pom2.xml`） | 安裝在 JIRA 端、供 BizDock 呼叫的 plugin |
| `bizdock-documentation` | – | 只有 README（各 repo 概覽） |
| `maf-desktop-community-packaging` | `com.agifac.maf.packaging` 12.1.1-SNAPSHOT | 舊的社群版 assembly 專案（已過時） |

建置順序：`app-framework` → `maf-desktop-datamodel` → `maf-desktop-app` → `maf-defaultplugins-extension` → `bizdock-packaging` → `bizdock-installation`。下游 POM 透過 `lib.app-framework.version`、`maf.maf-desktop-datamodel.version`、`play.app.version`、`maf.dbmdl.version` 等 property 引用上游版本。升版時這些 property 也要一起更新（`maf-desktop-app/development/tools/update_version.sh`）。

## 常用指令

本機建置時一律略過 GPG 簽章：

```bash
mvn -Dgpg.skip clean install        # 在任一元件內執行
```

開發 script 位於 `maf-desktop-app/development/tools/`。它們是 bash script，需要在 Linux 或 Git Bash 下執行。使用前先把 `env.cfg` 裡的 `BIZDOCK_GIT_ROOT` 設為本 repo 的根目錄路徑。

- `full_build.sh [-f|-m|-d]`：重建 framework＋model＋desktop（`-f`，預設）、model＋desktop（`-m`），或只重建 desktop（`-d`）。
- `restart_database.sh` / `stop_database.sh`：MariaDB container `development_bizdockdb`，位於 `127.0.0.1:3306`（database / user / password：`maf`/`maf`/`maf`）。
- `db_init.sh`：drop 資料庫，透過 replacer 執行兩個 migration package，再載入 `maf-desktop-app/conf/sql/init_base.sql` 和 `sample-data/init_data.sql`。範例 user 的 password 為 `pass1234`。
- `install_extensions.sh`：把預設 plugin extension 安裝到 `environment/maf-filesystem`。
- `mybatis/app_db_new.sh <name>` / `app_db_up.sh <env>`（以及 `framework_db_*`）：建立或套用 migration。

啟動應用程式：`cd maf-desktop-app && activator`，然後執行 `run 9000`。本機設定在 `conf/environment.conf` 和 `conf/framework.conf`，由 `application.conf` include 進來。

依環境 property 打包：
`mvn com.agifac.deploy:replacer-maven-plugin:replace -Dsource=<pkg>-<ver>.zip -Denv=my.properties` 會產生 `merged-<pkg>-<ver>.zip`。每個可部署元件在 `META-INF/com.agifac.deploy.replacer.resources.properties` 列出要替換的 template 檔，預設值放在 `src/main/properties/empty.properties`。

## 程式碼慣例（maf-desktop-app / app-framework）

- **Controller**（`app/controllers/{core,admin,api,dashboard,my,sso}`）繼承 `play.mvc.Controller`，透過 field `@Inject` 取得 service，並由 `conf/routes`（約 1,000 行）中的 `InjectedRoutesGenerator` 路由。授權使用 Deadbolt annotation（`@SubjectPresent`、`@Restrict(@Group(...))`、`@Dynamic(...)`），搭配 `app/security` 內的 `@With(Check*Exists.class)` action。permission 名稱定義在 `constants.IMafConstants`。
- **DAO**（`app/dao/<domain>/*Dao.java`）是 `abstract` class，使用 **static** `Finder<Long, T>` field 和 static query method（例如 `ActorDao.getActorById(id)`）。entity 採 soft delete，所以 query 通常會加上 `deleted=false` 條件。
- **Model** 繼承 `com.avaje.ebean.Model`，使用 public field、`@Version Timestamp lastUpdate` 和 `boolean deleted`。它們實作 `IModel`；透過 API 對外公開的 model 另外實作 `IApiObject`，並標註 `@JsonProperty`/`@ApiModelProperty`。
- **Service** 採 `IFooService` interface 加 `FooServiceImpl` implementation 的配對，在 Guice module 中 binding（`maf-desktop-app/app/modules/ApplicationServicesModule.java`，繼承 `framework.modules.FrameworkModule`）。
- form bean 放在 `app/utils/form/*FormData`。table 定義在 `app/utils/table` 和 `services/tableprovider`。view 是 `app/views` 內的 Twirl template。
- i18n：`conf/messages.{en,fr,de}`。新增的 key 必須三種語言都加上，透過 `framework.utils.Msg.get(...)` 取用。
- 每個檔案都有 GPL license header（可用 `development/copyright/fix.sh` 加上）。慣例是寫 Javadoc 並附 `@author` tag。Checkstyle（`conf/checkstyle.xml`）限制每行最多 160 字元。

## 跨模組架構重點

- **Security**：`app-framework/app/framework/security` 定義 `ISecurityService`（取得目前 user、`restrict(...)` 角色檢查、`dynamic(name, meta, id)` 動態權限）與 `AbstractSecurityServiceImpl`；desktop 的實作是 `maf-desktop-app/app/security/SecurityServiceImpl.java`，在其中以 `dynamicAuthenticationHandlers.put(IMafConstants.*_DYNAMIC_PERMISSION, ...)` 註冊每個動態權限，並委派給 `app/security/dynamic/*DynamicHelper`（controller 如 `PortfolioEntryController`、`SearchController`、`RoadmapController` 也會直接呼叫這些 helper 來過濾列表）。新增動態權限時，常數、handler、helper 三處都要改。
- **認證模式**：`conf/framework.conf` 的 `maf.authentication.mode`（`IFrameworkConstants.AuthenticationMode`：`STANDALONE` 由應用程式自行驗證；`FEDERATED`＝SAMLv2、`CAS_SLAVE`/`CAS_MASTER`＝CAS，皆透過 pac4j，`*_MASTER` 表示帳號佈建也由 BizDock 管理），另可啟用 `maf.authentication.bizdock_sso.*`（`framework.security.bizdock_sso`）。登入流程分派在 `AbstractAuthenticator`。
- **Framework service**：`app-framework/app/framework/services/*`（account、job、kpi、plugins、ext、storage、email、notification、custom_attribute、audit、router、script…），由 `framework.modules.FrameworkModule` binding；desktop 的 `ApplicationServicesModule` 繼承它並加入業務 service。
- **Plugin / Extension**：extension 是 jar + `conf/extension.xml`，由 `ExtensionManagerServiceImpl` 以 `PlayProxyClassLoader` 載入。`extension.xml` 內每個 `<plugin>` 指向一個 `IPluginRunner`（`start/stop`、`handleIn/OutProvisioningMessage`、menu / action descriptor），用 `<configuration-block>` 宣告預設設定，`<registration-configurator>` 宣告與 `PortfolioEntry` 等物件的註冊 UI；`<widget>` 宣告 dashboard widget（繼承 `WidgetController`）。plugin 的名稱/描述是 i18n key，要加進 extension 自己的 messages。
- **REST API**：`maf-desktop-app/app/controllers/api/{core,request,system}` 繼承 `ApiController`，每個 action 加 `@ApiAuthentication(additionalCheck = ApiAuthenticationBizdockCheck.class)` 與 Swagger 的 `@ApiOperation`。API 認證用簽章機制（`framework.services.api.server.ApiSignatureServiceImpl`），API application 與 key 在 admin UI（`controllers/admin/ApiManagerController`）管理。

## 資料庫 Migration

- script 位於 `maf-dbmdl/src/main/resources/repo/scripts/`（業務 table）和 `dbmdl-framework/src/main/resources/repo/scripts/`（framework table）。命名格式為 `YYYYMMDDhhmmss_<description>.sql`，版本彙整檔的名稱類似 `..._V17-2-0.sql`。
- 採用 MyBatis Migrations 格式：先寫正向 SQL，接著是 `-- //@UNDO` 區段。
- DDL 為 MySQL/MariaDB 語法，使用 `ENGINE=InnoDB DEFAULT CHARSET=latin1`。
- 修改 `maf-desktop-datamodel` 中的 Ebean entity 時，要在 `maf-dbmdl` 加上對應的 migration；如果是 `app-framework` 內的 `models.framework_models` entity，則加在 `dbmdl-framework`。

## 注意事項

- 部分 submodule 有先前建置留下的 `target/` 目錄，請勿修改或 commit。
- 以 CentOS 為基底的 Dockerfile，以及 CI 中 Oracle JDK / Activator 的下載網址，現在大多已失效，建置時可能需要手動處理 dependency。
