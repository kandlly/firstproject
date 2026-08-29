# intra-mart (iAP) 用户多组有效期间动态更新方案报告

## 一、 业务需求与背景

**历史版本差异**
在 intra-mart 早期版本（旧版架构）中，用户的有效期间通常作为用户表上的单一字段存在，因此可以直接通过执行整体的 `user.update` 来覆盖更新用户的有效期。

**当前版本架构与痛点**
在最新的 intra-mart Accel Platform (iAP) 的 **IM-共通マスタ（IM-Master）** 架构中，引入了期间化（Periodization）机制。用户主数据与有效期间（Term）变更为 **1:N** 的关系，支持存在多组不连续的有效期间（例如：`2026/05/01-2028/06/01` 以及 `2029/06/01-2030/06/01`），期间的空白段系统会自动判定为权限失效。
因此，传统的单一对象 Update 方式已无法满足需求。

**具体需求**
需要提供一种基于 Java API 的后端方案，能够接收前端页面传来的**动态有效期间列表（List）**，并安全、准确地更新用户的有效期间，同时保留原有的组织关联与主数据信息。

---

## 二、 解决方案设计思路

初版方案曾尝试"将新旧期间列表按开始时间排序后按下标一一对应"，但这个假设**只在纯追加/纯删末尾的场景下成立**：一旦业务上是在时间轴中间插入一条新期间，按下标对应会把某条旧期间的组织/角色绑定错误地套用到不相关的新期间上，且不会报错，属于静默数据错误，风险高于"全删全建"。

因此本版方案的核心策略调整为**"以 `termCd` 作为期间的显式身份标识，由前端明确告知每条期间是‘沿用哪个既有期间’还是‘全新期间’"**，具体分为以下几个关键步骤：

1. **显式身份识别**：前端传入的每条期间携带可选的 `termCd`。`termCd` 非空且能在数据库中查到，视为"沿用/修改既有期间"；`termCd` 为空，视为"全新期间"。不再依赖数组下标做隐式匹配。
2. **输入合法性校验**：对新期间列表本身做校验——每条期间 `startDate < endDate`，且列表内部彼此不重叠。非法输入在调用底层 API 之前直接拒绝，避免把不友好的底层异常暴露给调用方。
3. **提取主数据**：提取用户任一既有期间的基本信息（如姓名等 `IUser` 数据），用于新增期间的数据继承。
4. **前置清理（Delete）**：数据库中存在、但本次前端未再提交对应 `termCd` 的期间，视为要删除，优先执行 Delete，为时间轴腾出空位。
5. **安全顺序复用修改（Move）**：对于携带 `termCd` 的期间，调用 `moveTerm` 修改边界，**安全保留**该期间上挂载的组织架构（所属公司/部门）和角色关联。Move 的执行顺序不是简单的"从前往后"，而是每次挑选"目标区间当前不会与其他待处理期间的现有区间冲突"的一条先执行；如果出现循环冲突（A 想占 B 当前的位置、B 也想占 A 当前的位置），先把其中一条临时挪到一个不会冲突的占位日期上"打破死锁"，再继续处理，从而避免"扩张型"变更在移动过程中产生瞬时期间重叠。
6. **增量追加（Create）**：对于 `termCd` 为空的新增期间，调用 `createTerm` 进行全新创建，并注入步骤3中提取的用户基本信息。
7. **事务边界**：整个 Delete/Move/Create 序列应运行在同一个事务内，任一步骤失败都需要整体回滚，避免出现"部分期间已删、部分已改、部分未处理"的不一致状态（详见第五节待确认事项）。

---

## 三、 Java API 核心代码实现

基于 `jp.co.intra_mart.foundation.master.user.UserManager`，完整服务类代码如下（**方法名/签名尚未对照官方 SDK 逐一核实，详见第五节**）：

### 1. 数据传输对象 (DTO)

用于接收前端传入的时间段数据。相比初版，新增了可选的 `termCd` 字段，作为新旧期间的显式关联键。

```java
import java.util.Date;

public class UserTermDto {
    /** 既有期间的唯一标识；为 null 表示这是一条全新期间，需要走 createTerm */
    private String termCd;
    private Date startDate;
    private Date endDate;

    public UserTermDto() {}

    public UserTermDto(String termCd, Date startDate, Date endDate) {
        this.termCd = termCd;
        this.startDate = startDate;
        this.endDate = endDate;
    }

    public String getTermCd() { return termCd; }
    public void setTermCd(String termCd) { this.termCd = termCd; }
    public Date getStartDate() { return startDate; }
    public void setStartDate(Date startDate) { this.startDate = startDate; }
    public Date getEndDate() { return endDate; }
    public void setEndDate(Date endDate) { this.endDate = endDate; }
}
```

### 2. 动态更新服务类代码

```java
import jp.co.intra_mart.foundation.master.user.UserManager;
import jp.co.intra_mart.foundation.master.user.model.IUserBizKey;
import jp.co.intra_mart.foundation.master.user.model.UserBizKey;
import jp.co.intra_mart.foundation.master.user.model.IUser;
import jp.co.intra_mart.foundation.master.user.model.User;
import jp.co.intra_mart.foundation.master.common.model.ITerm;
import jp.co.intra_mart.foundation.master.common.model.Term;

import java.util.ArrayList;
import java.util.Calendar;
import java.util.Comparator;
import java.util.Date;
import java.util.HashMap;
import java.util.HashSet;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.UUID;

public class UserTermDynamicUpdateService {

    /**
     * 根据传入的动态时间段列表，更新用户的有效期间。
     * 注意：本方法内部连续执行 delete/move/create 多次底层调用，
     * 调用方必须保证本方法运行在同一个事务边界内（见方案报告第五节）。
     *
     * @param targetUserCd 目标用户CD
     * @param newTermList  前端传入的新有效期间列表（沿用期间需携带 termCd）
     */
    public void updateUserTermsDynamic(String targetUserCd, List<UserTermDto> newTermList) throws Exception {

        validateNewTermList(newTermList);

        UserManager userManager = new UserManager();
        IUserBizKey bizKey = new UserBizKey();
        bizKey.setUserCd(targetUserCd);

        // 1. 查询该用户当前所有期间，以 termCd 建立索引
        ITerm[] existingTermsArray = userManager.getTerms(bizKey);
        Map<String, ITerm> existingTermMap = new LinkedHashMap<>();
        if (existingTermsArray != null) {
            for (ITerm term : existingTermsArray) {
                existingTermMap.put(term.getTermCd(), term);
            }
        }

        // 2. 提取用户基础主数据（如姓名），用于新增期间的数据继承
        IUser baseUserInfo = new User();
        if (!existingTermMap.isEmpty()) {
            ITerm anyExisting = existingTermMap.values().iterator().next();
            IUser existUser = userManager.getUser(bizKey, anyExisting.getStartDate());
            if (existUser != null) {
                baseUserInfo = existUser;
            }
        }

        // 3. 按 termCd 是否能匹配到既有期间，拆分为"沿用修改"与"全新创建"两类
        List<UserTermDto> toUpdate = new ArrayList<>();
        List<UserTermDto> toCreate = new ArrayList<>();
        Set<String> keepTermCds = new HashSet<>();
        for (UserTermDto dto : newTermList) {
            if (dto.getTermCd() != null && existingTermMap.containsKey(dto.getTermCd())) {
                toUpdate.add(dto);
                keepTermCds.add(dto.getTermCd());
            } else {
                toCreate.add(dto);
            }
        }

        // 4. 前置清理：既有期间中，本次未再被提交 termCd 的，视为要删除
        //    优先执行，为后续 move 腾出时间轴空间
        for (Map.Entry<String, ITerm> entry : existingTermMap.entrySet()) {
            if (!keepTermCds.contains(entry.getKey())) {
                userManager.deleteTerm(bizKey, entry.getValue());
            }
        }

        // 5. 按安全顺序执行 move，避免"扩张型"变更在过程中产生瞬时期间重叠
        moveTermsSafely(userManager, bizKey, toUpdate, existingTermMap);

        // 6. 处理全新期间
        for (UserTermDto dto : toCreate) {
            ITerm targetTerm = new Term();
            targetTerm.setStartDate(dto.getStartDate());
            targetTerm.setEndDate(dto.getEndDate());
            targetTerm.setTermCd(generateTermCd());

            // createTerm 需传入 baseUserInfo 以防该期间下用户姓名等主数据丢失
            userManager.createTerm(bizKey, targetTerm, baseUserInfo);
        }
    }

    /**
     * 按"目标区间不与其他待处理期间的当前区间冲突"为条件，逐条挑选可安全执行的 move。
     * 若出现循环冲突，先把队首期间挪到一个不会冲突的临时占位日期，打破死锁后继续。
     */
    private void moveTermsSafely(UserManager userManager, IUserBizKey bizKey,
                                  List<UserTermDto> toUpdate, Map<String, ITerm> existingTermMap) throws Exception {

        // 各 termCd 当前的实时区间快照，随着 move 的执行而更新
        Map<String, ITerm> liveSnapshot = new HashMap<>();
        for (UserTermDto dto : toUpdate) {
            liveSnapshot.put(dto.getTermCd(), existingTermMap.get(dto.getTermCd()));
        }

        List<UserTermDto> pending = new ArrayList<>(toUpdate);
        int parkOffset = 0;
        int guard = pending.size() * pending.size() + 1; // 防止异常数据导致死循环

        while (!pending.isEmpty()) {
            UserTermDto movable = findNonConflicting(pending, liveSnapshot);

            if (movable == null) {
                // 循环冲突：先把队首期间挪去临时占位日期，释放它当前占用的空间
                UserTermDto parkTarget = pending.get(0);
                ITerm parkedTerm = buildParkingTerm(parkTarget.getTermCd(), parkOffset++);
                userManager.moveTerm(bizKey, liveSnapshot.get(parkTarget.getTermCd()), parkedTerm);
                liveSnapshot.put(parkTarget.getTermCd(), parkedTerm);
                continue;
            }

            ITerm targetTerm = new Term();
            targetTerm.setTermCd(movable.getTermCd());
            targetTerm.setStartDate(movable.getStartDate());
            targetTerm.setEndDate(movable.getEndDate());

            userManager.moveTerm(bizKey, liveSnapshot.get(movable.getTermCd()), targetTerm);
            liveSnapshot.put(movable.getTermCd(), targetTerm);
            pending.remove(movable);

            if (--guard <= 0) {
                throw new IllegalStateException("期间重排序超出预期迭代次数，可能存在无法收敛的冲突，请检查输入数据");
            }
        }
    }

    private UserTermDto findNonConflicting(List<UserTermDto> pending, Map<String, ITerm> liveSnapshot) {
        for (UserTermDto candidate : pending) {
            boolean conflict = false;
            for (UserTermDto other : pending) {
                if (other == candidate) {
                    continue;
                }
                ITerm otherLive = liveSnapshot.get(other.getTermCd());
                if (isOverlap(candidate.getStartDate(), candidate.getEndDate(), otherLive)) {
                    conflict = true;
                    break;
                }
            }
            if (!conflict) {
                return candidate;
            }
        }
        return null;
    }

    private boolean isOverlap(Date newStart, Date newEnd, ITerm other) {
        return newStart.before(other.getEndDate()) && other.getStartDate().before(newEnd);
    }

    /**
     * 生成一个远离正常业务范围、且相互之间也不重叠的临时占位期间。
     * ⚠️ 待确认：iAP 是否允许期间日期设置为如此极端的值，以及平台对日期取值范围的限制（见第五节）。
     */
    private ITerm buildParkingTerm(String termCd, int offset) {
        Calendar cal = Calendar.getInstance();
        cal.set(9000, Calendar.JANUARY, 1);
        cal.add(Calendar.DATE, offset * 2);
        Date parkStart = cal.getTime();
        cal.add(Calendar.DATE, 1);
        Date parkEnd = cal.getTime();

        ITerm parked = new Term();
        parked.setTermCd(termCd);
        parked.setStartDate(parkStart);
        parked.setEndDate(parkEnd);
        return parked;
    }

    /**
     * 校验新期间列表本身的合法性：开始日期早于结束日期，且列表内部彼此不重叠。
     */
    private void validateNewTermList(List<UserTermDto> newTermList) {
        if (newTermList == null || newTermList.isEmpty()) {
            throw new IllegalArgumentException("有效期间列表不能为空");
        }

        List<UserTermDto> sorted = new ArrayList<>(newTermList);
        sorted.sort(Comparator.comparing(UserTermDto::getStartDate));

        for (int i = 0; i < sorted.size(); i++) {
            UserTermDto term = sorted.get(i);
            if (term.getStartDate() == null || term.getEndDate() == null
                    || !term.getStartDate().before(term.getEndDate())) {
                throw new IllegalArgumentException("期间的开始日期必须早于结束日期，index=" + i);
            }
            if (i > 0) {
                UserTermDto prev = sorted.get(i - 1);
                if (term.getStartDate().before(prev.getEndDate())) {
                    throw new IllegalArgumentException(
                            "新期间列表内部存在重叠: " + prev.getTermCd() + " / " + term.getTermCd());
                }
            }
        }
    }

    /**
     * ⚠️ 待确认：termCd 的长度/格式/唯一性规则需对照 iAP 平台约束后调整，此处仅为示例实现。
     */
    private String generateTermCd() {
        return "TERM_" + UUID.randomUUID().toString().replace("-", "").substring(0, 15);
    }
}
```

---

## 四、 核心设计优势

1. **消除了错位风险**：不再依赖"排序后按下标对应"的隐式假设，而是以 `termCd` 作为期间的显式身份标识，中间插入新期间、删除中间期间等场景都不会导致组织/角色绑定被错误地套用到不相关的期间上。
2. **防重叠保护 (MasterException 防止)**：通过输入合法性校验、"先删后改"、以及 move 阶段"选无冲突项优先、循环冲突用临时占位打破死锁"的安全顺序算法，覆盖了"数量减少"和"期间扩张互相挤占"两类会触发时间交集错误的场景。
3. **关系链防断裂**：通过优先使用 `moveTerm` 而非粗暴的"全删全建"，最大程度保护了用户在特定期间内已经配置好的组织归属（所属公司、部门）和角色授权关系。
4. **动态适配能力**：无论前端传来的是 1 条还是 N 条期间记录，是纯修改、纯新增还是增删改混合，算法均能基于 `termCd` 自动分类处理，具备高度的通用性。
5. **失败可控**：输入校验提前到调用底层 API 之前，减少不可预期的底层异常；同时明确要求整个操作运行在事务边界内，避免部分成功导致的数据不一致。

---

## 五、 已知限制与待确认事项

以下几点在方案定稿、进入开发前需要对照 iAP 官方 API 文档 / 实际运行环境逐一核实，不能仅凭本报告直接实现：

1. **API 真实性**：`UserManager#getTerms/getUser/deleteTerm/moveTerm/createTerm`、`ITerm/Term`、`IUserBizKey/UserBizKey` 等具体类名、方法名、参数顺序和语义（尤其 `moveTerm` 是否真的会保留期间上挂载的组织/角色绑定），需要对照实际 SDK Javadoc 或反编译结果核实，本报告暂未做到 100% 确认。
2. **事务边界**：报告假设整个 delete/move/create 序列运行在同一事务内，但 iAP 侧具体由什么机制提供事务边界（容器声明式事务、平台自带的事务管理 API，还是需要显式引入 JTA `UserTransaction`），需要结合实际调用方式（Web API / Procedure / 批处理）确认，并在代码中显式接入。
3. **临时占位日期方案**：`buildParkingTerm` 用远超正常业务范围的日期（如 9000 年）作为循环冲突时的临时占位，用于打破"A 要占 B 当前位置、B 也要占 A 当前位置"这类死锁。需要确认平台对期间日期取值范围是否有限制，以及该极端日期是否会触发其他校验或影响 IM-共通マスタ 的期间空白判定逻辑。
4. **termCd 生成规则**：新增期间的 `termCd` 生成方式（长度、格式、唯一性约束）需对照平台规则调整，当前实现仅为示例。
5. **前端契约变更**：本版方案要求前端在提交"沿用修改"的期间时携带原有 `termCd`；如果前端现有实现无法提供该字段，需要先补充前端改造，否则退化为初版的"按下标猜测对应"方案，仍存在第一版报告中指出的错位风险。
