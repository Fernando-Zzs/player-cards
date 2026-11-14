# 球星卡面板

FC 风格的球星卡面板，一个静态页面，数据存在 Supabase，部署在 Vercel。

- 线上地址：https://player-cards-ten.vercel.app
- 页面代码：`index.html`（样式、脚本都在里面，无需构建）
- 推送到 `main` 分支后，Vercel 自动部署

## 数据

- Supabase 项目：`player-cards`（ap-southeast-1）
- 表 `public.cards`：每行一张卡，`data` 字段是卡片 JSON；所有人可读
- 表 `public.cards_history`：每次修改、删除前的旧版本，只有后台可见，用来恢复误删
- 表 `public.ratings`：评分，每人每张卡一票（主键 card_id + voter），所有人可读
- 表 `public.matches` / `match_players` / `mvp_votes`：比赛、上场名单和进球助攻、MVP 投票（每人每场一票）
- 卡面分数 = 所有人评分的平均；建卡时填的分数就是建卡人的那一票
- 写入只能通过函数：`save_card`、`delete_card`、`rate_card`、`save_match`、`delete_match`、`vote_mvp`

## 两种模式

`private.settings` 里的 `mode`：

- `open`（当前）：不用登录，每台设备生成一个随机密钥，按设备算一票。换设备或清缓存会变成新身份。
- `login`：邮箱验证码登录，按账号算一票。切换前要在 Supabase 后台配好发信（SMTP）和邮件模板（加 `{{ .Token }}`），并把 Site URL 设成线上地址。

```sql
update private.settings set value = 'login' where key = 'mode';  -- 切到登录模式
update private.settings set value = 'open'  where key = 'mode';  -- 切回无登录模式
```

## 管理员

管理员可以编辑、删除任何卡和比赛。open 模式下用管理员链接 `https://player-cards-ten.vercel.app/#admin=<口令>` 在设备上打开一次即可（口令只存哈希，不在代码里）。换口令：

```sql
update private.settings set value = extensions.crypt('新口令', extensions.gen_salt('bf')) where key = 'admin_token';
```

login 模式下加管理员：

```sql
insert into private.admins(user_id) select id from auth.users where email = '对方邮箱';
```

## 恢复误删的卡

```sql
-- 找到旧版本
select id, card_id, op, data->>'name' as name, at
from public.cards_history order by at desc limit 20;

-- 按 history 的 id 恢复
insert into public.cards(id, data)
select card_id, data from public.cards_history where id = 历史记录id
on conflict (id) do update set data = excluded.data;
```
