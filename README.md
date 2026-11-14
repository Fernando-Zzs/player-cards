# 球星卡面板

FC 风格的球星卡面板，一个静态页面，数据存在 Supabase，部署在 Vercel。

- 线上地址：https://player-cards-ten.vercel.app
- 页面代码：`index.html`（样式、脚本都在里面，无需构建）
- 推送到 `main` 分支后，Vercel 自动部署

## 数据

- Supabase 项目：`player-cards`（ap-southeast-1）
- 表 `public.cards`：每行一张卡，`data` 字段是卡片 JSON；所有人可读
- 表 `public.cards_history`：每次修改、删除前的旧版本，只有后台可见，用来恢复误删
- 表 `public.ratings`：评分，每人每张卡一票（主键 card_id + user_id），所有人可读
- 卡面分数 = 所有人评分的平均；建卡时填的分数就是建卡人的那一票
- 登录：Supabase Auth 邮箱验证码
- 写入只能通过函数：`save_card`（新建 / 编辑，只有建卡人或管理员能改）、`delete_card`、`rate_card`；未登录不能调用
- 管理员名单在 `private.admins`

## 加管理员

```sql
insert into private.admins(user_id)
select id from auth.users where email = '对方邮箱';
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
