---
title: beego orm多对多关系
date: 2019-04-19 13:54:51
tags: [go, beego]
categories: [develop]
---

今天聊聊beego中orm多对多关系的实现，我也是刚刚接触beego，大神莫要拍砖哦~_
系统中常用的权限管理为例，先看ER图:
![beego_m2m_er](/img/posts/beego_m2m_er.png)

每个用户对应一个角色，一个角色对应多种权限，一种权限也可分给多个角色,
获取某一用户所拥有的权限。

<!-- more -->

**方法1 自动生成中间表**
```
type RoleObject struct {
  Id     int           `json:"id"`
  Name   string        `json:"name"`
  Remark string        `json:"remark"`
  Perms  []*PermObject `orm:"rel(m2m);rel_table(tbl_role_permission)"` // rel_table指定中间表表名
}

func (a *RoleObject) TableName() string {
  return "tbl_role"
}

type PermObject struct {
  Id     int           `json:"id"`
  Name   string        `json:"name"`
  Remark string        `json:"remark"`
  Roles  []*RoleObject `orm:"reverse(many);column(role_id)"` // 设置一对多的反向关系
}

func (a *PermObject) TableName() string {
  return "tbl_permission"
}

type UserObject struct {
  Id      int         `json:"id"`
  Name    string      `json:"name"`
  Passwd  string      `json:"passwd"`
  Company string      `json:"companyt"`
  Email   string      `json:"email"`
  Phone   string      `json:"phone"`
  Role    *RoleObject `orm:"rel(fk)"`
}

func (a *UserObject) TableName() string {
  return "tbl_user"
}
```
`注意`由于beego orm的解析规则, 中间表tbl_role_permission中的字段名称会是`tbl_role_id`和`tbl_permission_id`。

**方法2 指定中间表字段**
```
type RoleObject struct {
  Id     int           `json:"id"`
  Name   string        `json:"name"`
  Remark string        `json:"remark"`
  Perms  []*PermObject `orm:"rel(m2m);rel_through(perf/user_demo/models.RolePermRel)"` // rel_through指定中间表结构
}

func (a *RoleObject) TableName() string {
  return "tbl_role"
}

type PermObject struct {
  Id     int           `json:"id"`
  Name   string        `json:"name"`
  Remark string        `json:"remark"`
  Roles  []*RoleObject `orm:"reverse(many);column(role_id)"` // 设置一对多的反向关系
}

func (a *PermObject) TableName() string {
  return "tbl_permission"
}

type RolePermRel struct {
  Id     int           `json:"id"`
  Role    *RoleObject    `orm:"rel(fk)"`
  Permission    *PermObject    `orm:"rel(fk)"`
}

func (a *RolePermRel) TableName() string {
  return "tbl_role_permission"
}

type UserObject struct {
  Id      int         `json:"id"`
  Name    string      `json:"name"`
  Passwd  string      `json:"passwd"`
  Company string      `json:"companyt"`
  Email   string      `json:"email"`
  Phone   string      `json:"phone"`
  Role    *RoleObject `orm:"rel(fk)"`
}

func (a *UserObject) TableName() string {
  return "tbl_user"
}
```

**获取用户的信息及其权限**
```
func Get_user_perm(userid int) (total int, data UserObject) {
  o := orm.NewOrm()
  err := o.QueryTable("tbl_user").Filter("id", userid).RelatedSel().One(&data)
  if err != nil {
      fmt.Println("query user failed, ", err)
  }
  // 必须调用LoadRelated载入模型的关系字段
  o.LoadRelated(data.Role, "perms")
  fmt.Println("User data: ", data)
  return
}
```

**查询出的结果**
```
{
  "data": {
    "id": 1,
    "name": "test",
    "passwd": "123456",
    "companyt": "ik",
    "email": "test@ik.com",
    "phone": "123456789",
    "Role": {
      "id": 1,
      "name": "admin",
      "remark": "manager for this sysstem",
      "Perms": [
        {
          "id": 1,
          "name": "UserControl.AddUser",
          "remark": "add user",
          "Roles": null
        },
        {
          "id": 2,
          "name": "UserControl.DelUser",
          "remark": "delet user",
          "Roles": null
        },
        {
          "id": 3,
          "name": "UserControl.UpdateUser",
          "remark": "update user",
          "Roles": null
        },
        {
          "id": 4,
          "name": "UserControl.QueryUser",
          "remark": "query user",
          "Roles": null
        }
      ]
    }
  },
  "total": 0
}
```

