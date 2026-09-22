# パスワード履歴未移行によるシステムエラー（WOE0001）対応 修正指示書

| 項目 | 内容 |
|---|---|
| 対象システム | WiseOfficePlus（intra-mart Accel Platform 上の Java Web アプリケーション） |
| 本書の目的 | 本事象の原因と修正内容を、実装担当者（または AI）がそのまま実装できる粒度で示す |

---

## 0. この指示書の使い方

- 「2. 事象」「3. 原因」は背景情報。実装だけ行う場合は **「5. 修正内容」** から読めばよい。
- **「4. 修正方針」で採用しなかった案** も記載している。勝手に別案へ変更しないこと。
- **「7. 実装前に確認が必要な事項」は実装前に必ず消化すること。** 未確認のまま実装すると、直ったように見えて直っていない可能性がある。
- 本書で明示していないファイル・設定は変更しないこと。特に **初回ログイン（`change-password-first-login` / `im_first_login`）に関する設定およびロジックは今回の対応範囲外であり、変更してはならない。**

---

## 1. 前提（確認済みの事実）

### 1.1 環境

| 項目 | 値 |
|---|---|
| アプリケーションサーバ | Resin v4.0（localhost:8080） |
| JDK | jdk-11.0.22_7 |
| コンテキストパス | `/wiseofficeplus` |
| 設定ファイル | `<リポジトリ>\conf\password-history.xml` |

### 1.2 `conf\password-history.xml`（group-default）

```xml
<group-default accessor-class="jp.wiseoffice.fw.security.StandardPasswordHistoryAccessorEx">
    <change-password-first-login>true</change-password-first-login>
    <password-expire-limit>0</password-expire-limit>
    <password-history-count>10</password-history-count>
    <password-expire-page>/user/password/expire</password-expire-page>
    <check-password enable="true">
        <check-password-length enable="true" min="8" max="50"/>
        <allow-latin-letters required="true">ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz</allow-latin-letters>
        <allow-number required="true">0123456789</allow-number>
        <allow-extra-char required="true">_-.+$#!/@</allow-extra-char>
        <deny-old-password>true</deny-old-password>
        <deny-userid>true</deny-userid>
    </check-password>
</group-default>
```

### 1.3 関連クラス

| クラス | パッケージ / 役割 |
|---|---|
| `StandardPasswordHistoryAccessorEx` | `jp.wiseoffice.fw.security`。`StandardPasswordHistoryAccessor`（intra-mart 標準）を **継承**。`password-history.xml` の `accessor-class` に設定済み |
| `PasswordHistoryNewArrivedProviderCustomize` | `NewArrivedProvider` 実装。TOP 画面「重要なお知らせ」ポートレットへパスワード有効期限の通知を供給する |
| `UserCertificationServletEx` | `UserCertificationServlet` を継承したログイン用サーブレット。閉局時間チェックおよび運用ログインルート判定を追加している |
| `PasswordHistoryManager` | intra-mart 標準。`jp.co.intra_mart.foundation.security.password` |

### 1.4 `StandardPasswordHistoryAccessorEx` の既存実装（本対応で参照する部分）

| 行 | メソッド | 内容 |
|---|---|---|
| L53 | クラス宣言 | `public class StandardPasswordHistoryAccessorEx extends StandardPasswordHistoryAccessor` |
| L644–L696 | `isPasswordExpired(groupId, userId, clientType, date)` | **オーバーライド済み。** L672 で `passwordExpireLimit > 0` のときのみ履歴を参照し、L680 で `history == null` の場合 `true` を返す（NPE 対策済み） |
| L708–L778 | `addPasswordHistory(groupId, createUserCd, userId, password)` | **Ex 独自の 4 引数オーバーロード。** L711 で履歴管理 OFF なら即 return、L722 で `passwordCryption.encrypt()`、L729–L743 で `SQL_ADD_PASSWORD_HISTORY` を INSERT、L771–L773 で `isFirstLogin` なら `setFirstLogin(false)`、L775–L777 で `verifyPasswordHistory()` による世代トリム |
| L1361–L1405 | `getPasswordHistoryCount(groupId, userId)` | 件数取得。該当行が無い場合 `0` を返す。**0 件でも例外・NPE は発生しない** |

---

## 2. 事象

1. ログイン後、TOP 画面の「重要なお知らせ」ポートレット描画時に `システムエラー（WOE0001）` が表示される。
2. URL は `http://localhost:8080/wiseofficeplus/certification`。GID / UCD は空欄。
3. サーバログ：

```
java.lang.NullPointerException
    ...
    StandardPasswordHistoryAccessor.java:398
```

4. DB の状態（`b_m_password_history`）：

```sql
select * from b_m_password_history
 where user_cd in ('tenant','tenant_1','tenant_2')
 order by record_date desc;
```

| USER_CD | PASSWORD | CREATE_USER_CD | CREATE_DATE |
|---|---|---|---|
| tenant_2 | GeEinT21iOs= | tenant | 2025/10/18 19:24:51 |
| tenant_1 | GeEinT21iOs= | tenant | 2025/10/18 19:24:51 |

**ログインユーザ `tenant` の行が存在しない。**

---

## 3. 原因

intra-mart のバージョンアップ（データ移行）時に `b_m_password_history` のデータが移行されなかった。

パスワード履歴を参照する処理が、履歴 0 件（取得結果が `null`）の状態を想定しておらず、`StandardPasswordHistoryAccessor` の 398 行目で `NullPointerException` が発生する。`StandardPasswordHistoryAccessorEx` が **オーバーライドしていないメソッド** で発生しているため、スタックトレースには親クラスのファイル名が出力される。

なお、履歴データの移行は **今回は実施しない** 方針が確定している。したがって本対応は「履歴が無ければログイン時に 1 件登録する」アプリケーション側の対応とする。

---

## 4. 修正方針

### 4.1 採用する方針

> **ログイン成功時、当該ユーザのパスワード履歴が 0 件であれば、入力された現在のパスワードで履歴を 1 件登録する。**
> 登録には intra-mart 標準 API（`PasswordHistoryManager`）を使用し、独自 SQL は書かない。

加えて、履歴を補完できないケース（後述 7.4）でも画面が落ちないよう、ポートレット側の null チェック順序を是正する。

### 4.2 採用しない方針（実施しないこと）

| 案 | 不採用の理由 |
|---|---|
| 旧環境から `b_m_password_history` をエクスポート／インポートする | 履歴は移行しない方針が確定しているため |
| `INSERT INTO b_m_password_history ...` の独自 SQL を新規に書く | パスワードの暗号化方式（`passwordCryption`）を標準実装と一致させる必要があり、独自実装は破綻リスクが高い。標準 API を使う |
| 初回ログインフラグ（`im_first_login`）を操作するコードを追加する | 今回の対応範囲外。追加しないこと（ただし 6.1 の副作用に注意） |
| NPE を try-catch で握り潰すだけの対応 | 履歴が登録されないため、パスワード変更時の世代チェック（`deny-old-password`）が機能しない状態が残る |

---

## 5. 修正内容

修正は 4 点。**修正 1 → 2 → 3 の順に実施すること。修正 4 は独立して実施可。**

---

### 修正 1（新規）：パスワード履歴補完クラスの追加

**新規ファイル**：`jp/wiseoffice/fw/security/PasswordHistoryInitializer.java`

```java
package jp.wiseoffice.fw.security;

import org.apache.commons.lang3.StringUtils;

import jp.co.intra_mart.foundation.security.password.PasswordHistoryException;
import jp.co.intra_mart.foundation.security.password.PasswordHistoryManager;

/**
 * パスワード履歴の初期登録ユーティリティ。<br/>
 * <p>
 * バージョンアップ時に b_m_password_history が移行されなかった環境において、
 * パスワード履歴が 0 件のユーザに対して初期履歴を 1 件登録する。
 * </p>
 */
public final class PasswordHistoryInitializer {

    /** ロガー。※プロジェクトの既存ロガー実装に合わせること（6.3 参照） */
    private static final Logger logger =
            Logger.getLogger(PasswordHistoryInitializer.class);

    private PasswordHistoryInitializer() {
        // インスタンス化禁止
    }

    /**
     * 指定ユーザのパスワード履歴が 0 件の場合のみ、初期履歴を 1 件登録する。<br/>
     * <br/>
     * 本メソッドは補完処理であり、失敗してもログイン処理を中断させない。<br/>
     *
     * @param userCd   認証済みのログインユーザコード
     * @param password 平文パスワード（ログイン画面で入力された値）
     */
    public static void initializeIfEmpty(final String userCd, final String password) {

        // ユーザコードまたはパスワードが取得できない場合は何もしない。
        // （運用ログインルート等、平文パスワードが存在しない経路が該当する）
        if (StringUtils.isBlank(userCd) || StringUtils.isBlank(password)) {
            return;
        }

        try {
            final PasswordHistoryManager manager = new PasswordHistoryManager();

            // 履歴管理を行わない設定（password-history-count = 0）の場合は何もしない
            if (!manager.isPasswordHistoryManaged()) {
                return;
            }

            // 既に履歴が存在する場合は何もしない
            if (manager.getPasswordHistoryCount(userCd) > 0) {
                return;
            }

            // 履歴 0 件 → 現在のパスワードで初期履歴を 1 件登録する
            manager.addPasswordHistory(userCd, password);

            logger.info("パスワード履歴が存在しないため初期履歴を登録しました。userCd=" + userCd);

        } catch (final PasswordHistoryException e) {
            // 補完処理の失敗はログインを止めない
            logger.warn("パスワード履歴の初期登録に失敗しました。userCd=" + userCd, e);

        } catch (final RuntimeException e) {
            // NullPointerException 等も同様にログインを止めない
            logger.warn("パスワード履歴の初期登録で想定外のエラーが発生しました。userCd=" + userCd, e);
        }
    }
}
```

#### 実装上の注意

- **件数判定には `getPasswordHistoryCount()` を使うこと。** `getLatestPasswordHistory()` は 0 件時に `null` を返す、あるいは今回の NPE の発生元そのものである可能性があり、判定に使ってはならない。
- `catch (RuntimeException)` を省略しないこと。`addPasswordHistory()` 内部で NPE が発生した場合でもログインを継続させるための保険である。
- `StringUtils` は既存プロジェクトで使用している実装（`org.apache.commons.lang3` か `jp.co.intra_mart.common.aid.jdk.java.lang.StringUtil` か）に合わせること。`UserCertificationServletEx` が既に import しているものを踏襲する。

---

### 修正 2（既存改修）：`UserCertificationServletEx` から補完処理を呼び出す

**対象ファイル**：`UserCertificationServletEx.java`

`doGet` と `doPost` の **2 箇所** に同じ処理を追加する。

#### 2-a. `doPost`（現状 L168 付近）

**修正前**

```java
        try {
            super.doPost(request, response);
        } catch (ServletException | IOException e) {
            authLog.logWrite(loginInfo, gwId, AuthLogOutEx.RESULT_NG);
        }

        // ログ出力
        authLog.logWrite(loginInfo, gwId, AuthLogOutEx.RESULT_OK);
```

**修正後**

```java
        try {
            super.doPost(request, response);
        } catch (ServletException | IOException e) {
            authLog.logWrite(loginInfo, gwId, AuthLogOutEx.RESULT_NG);
        }

        // パスワード履歴の初期登録（履歴未移行環境への対応）
        // 認証後のユーザコードを再取得する。loginInfo.getUserCd() は認証前に
        // 設定された値のため、ここでは使用しない。
        final String authenticatedUserCd =
                ((AccountContext) Contexts.get(AccountContext.class)).getUserCd();
        PasswordHistoryInitializer.initializeIfEmpty(
                authenticatedUserCd, request.getParameter(this.passwordKey));

        // ログ出力
        authLog.logWrite(loginInfo, gwId, AuthLogOutEx.RESULT_OK);
```

#### 2-b. `doGet`（現状 L117 付近）

`doPost` と同一の処理を、`super.doGet(request, response);` の try-catch 直後、`// ログ出力` の直前に追加する。

#### 実装上の注意

- **呼び出し位置を `super.doXxx()` より前にしてはならない。** 認証前に履歴を登録してしまう。
- `AccountContext` / `Contexts` は同ファイル内で既に使用されているため、追加の import は不要（要確認）。
- **認証失敗時に補完されないことの確認が必要。** `super.doPost()` が例外を投げずに認証エラー画面へ forward するケースでは、認証後の `AccountContext.getUserCd()` が空またはゲストユーザになるはずである。空であれば `initializeIfEmpty()` 側の `isBlank` 判定で抜けるため問題ないが、**ゲストユーザコードが返る実装の場合は明示的な除外が必要**。実装前にログ出力で実値を確認すること（7.3 参照）。

---

### 修正 3（既存改修・予備案）：`StandardPasswordHistoryAccessorEx` に 3 引数版を追加

> **この修正は、修正 1・2 を入れても履歴が登録されない場合、または `addPasswordHistory` で NPE が再発する場合にのみ実施する。**
> 先に修正 1・2 だけで動作確認を行い、正常に登録されるなら本修正は不要。

`PasswordHistoryManager#addPasswordHistory(userCd, password)` は、内部でアクセサの **3 引数版** `addPasswordHistory(groupId, userCd, password)` を呼び出す。`StandardPasswordHistoryAccessorEx` は 4 引数版のみを独自定義しているため、3 引数版は intra-mart 標準実装が動作する。

標準実装側で問題が起きる場合は、以下を `StandardPasswordHistoryAccessorEx` に追加し、Ex の 4 引数版へ委譲させる。

```java
    /**
     * パスワード履歴の追加（3 引数版）。<br/>
     * <br/>
     * 標準実装ではなく、本クラスの 4 引数版へ委譲する。<br/>
     *
     * @param groupId  グループID
     * @param userId   ユーザID
     * @param password 入力パスワード
     * @throws PasswordHistoryException パスワード履歴の追加に失敗した場合にスローされます。
     */
    @Override
    public void addPasswordHistory(final String groupId, final String userId,
            final String password) throws PasswordHistoryException {

        addPasswordHistory(groupId, userId, userId, password);
    }
```

#### 実装上の注意（重要）

- **本修正はパスワード変更画面など、既存のすべてのパスワード履歴追加処理の挙動を変える。** 影響範囲が広いため、予備案の位置づけとしている。
- 追加前に、`StandardPasswordHistoryAccessorEx` に **3 引数版 `addPasswordHistory` が既に定義されていないか** を必ず確認すること。定義済みの場合、本修正は不要かつ有害。
- 第 2 引数 `createUserCd` に `userId` を渡している。標準実装が別の値（操作者のユーザコード等）を設定している場合、`b_m_password_history.CREATE_USER_CD` の値が従来と変わる。許容できるかを確認すること。

---

### 修正 4（既存改修）：ポートレットの null チェック順序を是正

**対象ファイル**：`PasswordHistoryNewArrivedProviderCustomize.java`
**対象メソッド**：`getNewArrivedList(String user, String group)`

現状、L93 の `mgr.isPasswordExpired(...)` が、L97–L104 の null チェックより **先に** 実行されている。`isPasswordExpired` は検査例外を宣言していないため、アクセサ内部で `NullPointerException` が発生した場合、そのまま呼び出し元へ伝播し、L102 の既存 null 対策に到達しない。

履歴を補完できないユーザ（7.4 参照）に対する保険として、null チェックを前倒しする。

**修正前（L89–L104）**

```java
        // 今日の日付から、指定した日数後の日付でパスワード期限切れになるかどうかチェックする。
        final Calendar calendar = Env.getSystemDate();
        calendar.add(Calendar.DATE, limitDays);
        if (!mgr.isPasswordExpired(user, SystemClientType.getDefaultClientTypeId(),
                calendar.getTime())) {
            return new NewArrived[0];
        }

        PasswordHistory passwordHistory = null;
        try {
            passwordHistory = mgr.getLatestPasswordHistory(user);
        } catch (final PasswordHistoryException e1) {
        }
        if (passwordHistory == null) {
            return new NewArrived[0];
        }
```

**修正後**

```java
        // パスワード履歴が存在しない場合は通知対象外とする。
        // （履歴未移行環境への対応。isPasswordExpired より先に判定すること）
        PasswordHistory passwordHistory = null;
        try {
            passwordHistory = mgr.getLatestPasswordHistory(user);
        } catch (final PasswordHistoryException e1) {
        }
        if (passwordHistory == null) {
            return new NewArrived[0];
        }

        // 今日の日付から、指定した日数後の日付でパスワード期限切れになるかどうかチェックする。
        final Calendar calendar = Env.getSystemDate();
        calendar.add(Calendar.DATE, limitDays);
        if (!mgr.isPasswordExpired(user, SystemClientType.getDefaultClientTypeId(),
                calendar.getTime())) {
            return new NewArrived[0];
        }
```

#### 実装上の注意

- **ブロックの順序を入れ替えるだけ**であり、ロジックそのものは変更しない。
- L105 以降（`// 期限日を作成` 以降）は一切変更しない。
- `passwordHistory` 変数の宣言位置が前に移動するため、L107 の `passwordHistory.getDate()` はそのまま動作する。

---

## 6. 副作用・注意事項

### 6.1 初回ログインフラグが落ちる（仕様として発生）

`StandardPasswordHistoryAccessorEx#addPasswordHistory`（L771–L773）は以下を実行する。

```java
        if (isFirstLogin(groupId, userId)) {
            setFirstLogin(groupId, userId, false);
        }
```

`change-password-first-login` は `true` 設定のため、**初回ログインフラグが立っているユーザは、履歴補完と同時にフラグが `false` になり、初回パスワード変更要求が出なくなる。**

本件は標準 API の仕様であり、今回の対応方針（初回ログイン関連は変更しない）に従い **対策コードは追加しない**。本書の読者は、この副作用を認識したうえで実装すること。

> 許容できない場合のみの任意対応：補完前に `manager.isFirstLogin(userCd)` を退避し、補完後に `manager.setFirstLogin(userCd, 退避値)` で戻す。**現時点では実施しない。**

### 6.2 `deny-old-password` との関係

`deny-old-password` が `true` のため、補完した「現在のパスワード」が履歴に載る。結果として、**補完後の初回パスワード変更で、現在と同じパスワードを再設定できなくなる**。これは本来の仕様どおりの挙動である。

### 6.3 ロガー

サンプルコードの `Logger` は仮置きである。`StandardPasswordHistoryAccessorEx` / `UserCertificationServletEx` で使用しているロガー実装（intra-mart 標準 `jp.co.intra_mart.common.platform.log.Logger` か SLF4J か）に合わせること。

### 6.4 同時ログイン

`getPasswordHistoryCount()` による判定と `addPasswordHistory()` の間に排他制御は無い。同一ユーザが同時に複数回ログインした場合、履歴が 2 件登録される可能性がある。`password-history-count` は 10 のため実害は無く、許容とする。

---

## 7. 実装前に確認が必要な事項

### 7.1 `password-expire-limit` の実機値【必須】

リポジトリの `conf\password-history.xml` では `<password-expire-limit>0</password-expire-limit>`（無期限）である。この値が 0 の場合、

- `PasswordHistoryNewArrivedProviderCustomize` は L72 の `if (mgr.getPasswordExpireLimit() <= 0) return new NewArrived[0];` で即座に空配列を返す
- `StandardPasswordHistoryAccessorEx#isPasswordExpired` も L672 の `if (passwordExpireLimit > 0)` に入らず `false` を返す

つまり **この設定のままであれば、いずれも履歴を参照しない**。事象が再現している以上、実機の値または適用される設定が異なる可能性が高い。

確認すること：

1. 実機の `production/webapp/default/wiseofficeplus/WEB-INF/conf/password-history.xml` の `password-expire-limit` の値
2. `group-default` 以外に `<group>` 要素による個別設定が存在しないか
3. 上記が 0 であった場合、NPE は別の経路で発生していることになるため、7.2 を優先して特定すること

### 7.2 `StandardPasswordHistoryAccessor.java:398` のメソッド特定【必須】

`StandardPasswordHistoryAccessorEx` がオーバーライドしていないメソッドで発生している。以下のいずれかで特定すること。

- Eclipse に intra-mart のソース JAR をアタッチして 398 行目を開く
- 逆コンパイラ（Enhanced Class Decompiler 等）で `StandardPasswordHistoryAccessor.class` を開く
- 完全なスタックトレースをログから取得し、呼び出し元メソッドを確認する

特定できたメソッドが履歴 0 件で NPE を起こすことを確認し、修正 1・2 による補完後に正常動作することを確認する。

### 7.3 認証失敗時のユーザコード【必須】

`super.doPost()` の後に `((AccountContext) Contexts.get(AccountContext.class)).getUserCd()` が返す値を、**認証成功時と認証失敗時の両方**でログ出力して確認すること。認証失敗時に実在のユーザコードが返る場合、修正 2 に明示的な認証結果判定を追加する必要がある。

### 7.4 運用ログインルートの扱い【要判断】

`UserCertificationServletEx#isUnyouLoginRoute()` が `true` となる経路（Cookie + HMAC による自動ログイン）では、`request.getParameter(this.passwordKey)` が `null` となるため、**平文パスワードが取得できず履歴を補完できない**。

この経路でログインするユーザについては、修正 4（ポートレットの null チェック順序是正）により画面は落ちなくなるが、履歴は登録されないままとなる。

以下のいずれかを選択すること。

- **A**：許容する（修正 4 の保険で画面は落ちないため）
- **B**：この経路のユーザには別途 DB へ直接初期履歴を投入する（別作業）

### 7.5 `StandardPasswordHistoryAccessorEx` の既存 3 引数版の有無

修正 3 を実施する場合のみ。同クラス内に `addPasswordHistory(String, String, String)` が既に定義されていないかを検索して確認すること。

---

## 8. 動作確認手順

テスト環境で実施すること。

| # | 手順 | 期待結果 |
|---|---|---|
| 1 | 対象ユーザの履歴を削除<br>`delete from b_m_password_history where user_cd = 'tenant';` | 0 件になる |
| 2 | 当該ユーザでログイン | システムエラー（WOE0001）が出ず、TOP 画面が正常表示される |
| 3 | `select * from b_m_password_history where user_cd = 'tenant';` | **1 件** 登録されている。`PASSWORD` 列が他ユーザと同じ形式（Base64 文字列）である |
| 4 | ログアウトし、再度ログイン | 履歴が **1 件のまま**（重複登録されない） |
| 5 | パスワード変更画面から、現在と同じパスワードへの変更を試みる | `deny-old-password` によりエラーとなる（6.2 の想定どおり） |
| 6 | パスワード変更画面から、別のパスワードへ変更する | 正常に変更でき、履歴が 2 件になる |
| 7 | 既に履歴が存在するユーザ（`tenant_1`）でログイン | 履歴件数が変化しない。TOP が正常表示される |
| 8 | 運用ログインルート経由でログイン | システムエラーが出ない（履歴は登録されなくてよい。7.4 参照） |
| 9 | 誤ったパスワードでログイン失敗 | 履歴が登録されない |

---

## 9. 影響範囲

| ファイル | 変更種別 | 影響 |
|---|---|---|
| `PasswordHistoryInitializer.java` | 新規 | 無し（新規クラス） |
| `UserCertificationServletEx.java` | 改修（2 箇所追加） | ログイン処理。補完処理は try-catch で保護しており、失敗してもログインは継続する |
| `PasswordHistoryNewArrivedProviderCustomize.java` | 改修（ブロック順序入替） | TOP 画面「重要なお知らせ」ポートレットのみ。ロジック変更なし |
| `StandardPasswordHistoryAccessorEx.java` | 改修（修正 3 実施時のみ） | **パスワード履歴追加処理全般。** 実施時は影響調査を別途行うこと |

### ロールバック

修正 1・2・4 はいずれも既存ロジックを書き換えていない（追加および順序入替のみ）ため、該当箇所を元に戻すだけで復旧できる。DB へ登録済みの初期履歴を消す場合は、`CREATE_DATE` で対象行を特定して削除する。

---

## 10. 参考資料

- [PasswordHistoryManager（intra-mart Javadoc）](https://api.intra-mart.jp/iap/javadoc/platform-all-dev_apidocs/jp/co/intra_mart/foundation/security/password/PasswordHistoryManager.html)
- [StandardPasswordHistoryAccessor（intra-mart Javadoc）](https://api.intra-mart.jp/iap/javadoc/all-dev_apidocs/jp/co/intra_mart/foundation/security/password/StandardPasswordHistoryAccessor.html)
- [PasswordHistoryNewArrivedProvider（intra-mart Javadoc）](https://api.intra-mart.jp/iap/javadoc/all-dev_apidocs/jp/co/intra_mart/foundation/security/password/PasswordHistoryNewArrivedProvider.html)
- [パスワード履歴管理設定（設定ファイルリファレンス）](https://document.intra-mart.jp/library/iap/public/configuration/im_configuration_reference/texts/im_tenant/password-history-config/index.html)
