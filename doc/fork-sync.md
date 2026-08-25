# Fork 定制与上游同步

本仓库是 [maillab/cloud-mail](https://github.com/maillab/cloud-mail) 的 fork。为了让「自己的改动」和「上游更新」互不干扰,定制代码单独放在补丁分支上,`main` 只做上游镜像。

## 分支模型

| 分支 | 用途 | 约定 |
| --- | --- | --- |
| `upstream/main` | 上游 `maillab/cloud-mail` | 只读 |
| `main` | 上游镜像 | **只做快进合并,不在上面提交任何自己的改动** |
| `feat/account-desc-order` | 定制补丁:邮箱列表倒序 | 所有定制提交都落在这里 |

远程配置:

```
origin     git@github.com:up-jack-ma/cloud-mail.git
upstream   https://github.com/maillab/cloud-mail.git
```

如果 `upstream` 不存在,补一次即可:

```bash
git remote add upstream https://github.com/maillab/cloud-mail.git
```

## 本分支的定制内容

共 3 个文件,同步时冲突只可能出现在这几处。

### `mail-worker/src/service/account-service.js`

- `list()`:排序由 `sort DESC, accountId ASC` 改为 `sort DESC, accountId DESC`;首页游标默认值由 `0` 改为 `9999999999`;keyset 分页条件由 `gt(accountId, cursor)` 改为 `lt(accountId, cursor)`。

  三处必须一起改。只翻转 `orderBy` 而不动游标,翻到第二页会查不到数据。

- `setAsTop()`:改为把目标账号的 `sort` 置为该用户的 `max(sort) + 1`,不再修改主账号的 `sort`。

  上游实现是「把主账号 sort 顶高,目标账号排它下面」,隐含「主账号永远第一」。倒序之后主账号(创建最早、id 最小)本应沉到底部,保留上游逻辑会导致一执行置顶就把主账号拉回顶部。

### `mail-vue/src/layout/account/index.vue`

- `submit()`:新增邮箱由 `accounts.push()` 改为插入到 `sort === 0` 区段的最前面,并调用 `changeAccount()` 选中它,右侧邮件列表随之切换。插到 `sort === 0` 之前而不是整个列表最前面,是为了不越过已置顶的账号。
- `getAccountList()`:首屏加载完成后 `changeAccount(list[0])`。上游只设了 `accountStore.currentAccount` 对象而没设 `currentAccountId`,两者会指向不同账号。
- `changeUserAccountName` 监听:由 `accounts[0].name` 改为按 `accountId` 查找。主账号不再固定在下标 0,而且可能还没被分页加载出来,所以要判空。
- `setAsTop()`:由 `accounts.splice(1, 0, item)` 改为 `unshift`,并同步更新本地 `item.sort`,与后端保持一致。

### `mail-vue/src/init/init.js`

启动时用 `accountList(0, 1)` 取最新账号作为默认选中项(受 `hasPerm('account:query')` 保护,失败则回退到主账号)。

不加这段也能工作 —— 账号列表加载完会触发 `currentAccountId` 变化 —— 但会先加载主账号邮件再跳到最新账号,每次进页面都有一次可见的闪烁和一次多余请求。

## 同步上游

### 1. 把 main 快进到上游

```bash
git fetch upstream
git checkout main
git merge --ff-only upstream/main
git push origin main
```

`--ff-only` 是刻意的:如果这一步失败,说明有人往 `main` 上提交了东西,先把它挪走再同步。

### 2. 把上游变更合进定制分支

```bash
git checkout feat/account-desc-order
git merge main
```

有冲突就解,冲突范围参照上面「本分支的定制内容」。解完:

```bash
git push origin feat/account-desc-order
```

### 3. 冲突处理要点

- **`account-service.js` 的 `list()`**:确认合并后 `orderBy` 是 `desc(sort), desc(accountId)`、游标默认值是 `9999999999`、条件是 `lt(accountId, ...)`。三者缺一分页就会错。
- **`account-service.js` 的 `setAsTop()`**:如果上游重写了这个方法,以「目标账号 = `max(sort) + 1`,不动主账号」为准。
- **`account/index.vue`**:上游若调整了新增/刷新逻辑,保证两件事即可 —— 新账号插在 `sort === 0` 区段最前面,以及首屏 `changeAccount(list[0])`。
- **`init.js`**:上游若重构了 `init()`,把那段 `hasPerm('account:query')` 判断挪到 `userStore.user` 赋值之后就行(`hasPerm` 依赖它)。

### 4. 验证

`mail-worker` 端的分页可以不起服务直接用 sqlite 验证:

```bash
sqlite3 /tmp/t.db <<'SQL'
CREATE TABLE account(account_id INTEGER PRIMARY KEY AUTOINCREMENT, email TEXT, user_id INT, sort INT DEFAULT 0, is_del INT DEFAULT 0);
INSERT INTO account(email,user_id,sort) VALUES
 ('admin@x',1,0),('a1@x',1,0),('a2@x',1,0),('a3@x',1,0),('a4@x',1,0),
 ('a5@x',1,0),('a6@x',1,0),('pinned@x',1,3);
SELECT account_id,email,sort FROM account WHERE user_id=1 AND is_del=0
 AND (sort < 9999999999 OR (sort = 9999999999 AND account_id < 9999999999))
 ORDER BY sort DESC, account_id DESC LIMIT 3;
SQL
```

期望第一页是 `pinned(8) → a6(7) → a5(6)`;把游标换成上一页最后一条的 `(sort, account_id)` 继续翻,应当不重不漏,主账号 `admin(1)` 排在最后。

前端跑一遍:

```bash
cd mail-vue && pnpm install && pnpm build
```

## 部署

CI(`.github/workflows/deploy-cloudflare.yml`)的触发条件是 `push` 到 `main`,定制分支推上去**不会**自动部署。工作流带 `workflow_dispatch`,所以:

> GitHub → Actions → `🚀 Deploy cloud-mail to Cloudflare Workers` → Run workflow → 分支选 `feat/account-desc-order`

这样不用改任何上游文件。

如果嫌手动麻烦,另一种做法是反过来:`main` 当部署分支,定制分支只当补丁存档,同步时在 `main` 上 `git merge upstream/main` 解一次冲突。代价是 `main` 不再是干净镜像。

## 数据注意事项

如果在改动之前用过「置顶」,主账号的 `sort` 已被写成非 0,倒序后它仍会排在最上面。清一次即可:

```sql
UPDATE account SET sort = 0;
```

## 备注

`origin/dev-desc` 是更早一次定制的分支,与当前 `main` 属于两段无关历史(`git merge-base` 无输出),已无法合并,可以删除。
