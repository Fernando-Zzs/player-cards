# 球星卡面板

FC 风格的球星卡面板，一个静态页面，数据存在 Supabase，部署在 Vercel。

- 线上地址：https://player-cards-ten.vercel.app
- 页面代码：`index.html`（样式、脚本都在里面，无需构建）
- 推送到 `main` 分支后，Vercel 自动部署

## 数据

- Supabase 项目：`player-cards`（ap-southeast-1）
- 表 `public.cards`：每行一张卡，`data` 字段是卡片 JSON；所有人可读
- 表 `public.cards_history`：每次修改、删除前的旧版本，只有后台可见，用来恢复误删
- 写入只能通过带口令的函数：`save_card`、`delete_card`，口令校验用 `check_code`

## 换编辑口令

在 Supabase SQL Editor 里执行（把新口令换进去）：

```sql
update private.settings
set value = extensions.crypt('新口令', extensions.gen_salt('bf'))
where key = 'edit_code';
```

已经输过旧口令的设备，下次保存时会提示口令不对，重新输入即可。

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
