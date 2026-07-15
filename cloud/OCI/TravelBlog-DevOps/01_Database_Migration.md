# 数据库迁移

**Date**
2026-07-15

---

## 目标：
  - 用MySQL代替SQLite
  - 尽量保留原有的用户数据

---

## 过程

### 1.本地准备

#### 1.1 requirements.txt里添加PyMySQL

#### 1.2 改Flask配置文件
  进入 app/config.py，把写死的连接SQLite:
  ```
  SQLALCHEMY_DATABASE_URI = f"sqlite:///{BASE_DIR / 'app.db'}"
  ```
  改成本地开发环境没有`DATABASE_URL` → 自动用 SQLite，服务器在`.env`设置了`DATABASEL_URL` → 自动用MySQL：
  ```
  SQLALCHEMY_DATABASE_URI = os.environ.get(
    "DATABASE_URL",
    f"sqlite:///{BASE_DIR / 'app.db'}"
  )
  ```
  
