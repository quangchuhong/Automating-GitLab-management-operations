# GitLab Infrastructure as Code cho Tổ Chức Phòng Ban

Tài liệu này hướng dẫn triển khai một luồng **tự động hoá** trên GitLab để:

- Tạo **groups / subgroups** theo từng phòng ban
- Tạo **projects**
- Tạo hoặc đồng bộ **users**
- **Phân quyền** (Team Lead, Admin sub-group, Developer)
- Tất cả được điều khiển qua **file YAML + Merge Request**

---

## 1. Mục tiêu & Kiến trúc tổng quan

### 1.1. Mục tiêu

- Chuẩn hoá cấu trúc GitLab theo phòng ban:
  - `cloudops`, `devops`, `appops`, `developer`, `tester`, `db`
- Mỗi phòng có:
  - **Root group** (ví dụ: `devops`)
  - Các **sub-groups** (ví dụ: `devops/ci-cd-projects`)
  - Bên trong mỗi sub-group có thể tạo **nhiều projects** và sub-group con (optional)
- Phân quyền:
  - Team Lead: **Owner** trên root group (quyền cao nhất trong group đó)
  - Admin sub-group: **Maintainer** (review, merge code, quản lý project trong subgroup)
  - Thành viên còn lại: **Developer** (push code, tạo branch, mở MR, không chỉnh setting)

### 1.2. Kiến trúc giải pháp

Sử dụng 1 repo trung tâm để quản lý cấu trúc GitLab bằng code:

- Repo: `gitlab-admin/infra-gitlab-config` (tên gợi ý, có thể đổi)
- Bên trong repo:
  - `config/organizations.yml` – Định nghĩa phòng ban, groups, projects, users, roles
  - `scripts/manage_gitlab.py` – Script Python đọc YAML, gọi GitLab API để:
    - Tạo group/subgroup/project
    - Tạo user (nếu chưa tồn tại)
    - Gán role cho user
  - `.gitlab-ci.yml` – Pipeline GitLab CI:
    - Validate cấu hình khi tạo Merge Request
    - Áp dụng thay đổi sau khi MR được merge

**Luồng hoạt động:**

1. Người dùng / PM cập nhật `config/organizations.yml` (thêm user, project, subgroup, …).
2. Tạo **Merge Request** vào branch `main`.
3. CI job `validate-config` kiểm tra file YAML.
4. Team Lead/Admin review MR:
   - Nếu OK → merge vào `main`.
5. CI job `apply-config` chạy:
   - Script `manage_gitlab.py` gọi GitLab API và cập nhật GitLab theo config.

---

## 2. Chuẩn bị môi trường

### 2.1. Yêu cầu GitLab

- GitLab **Self-Hosted** hoặc GitLab.com đều được.
- Cần một **Admin user** trên GitLab để:
  - Tạo user mới qua API
  - Quản lý group, project

### 2.2. Tạo Personal Access Token (Admin)

1. Đăng nhập bằng tài khoản **Admin**.
2. Vào: **User Settings → Access Tokens** (hoặc tương đương).
3. Tạo token với quyền:
   - `api`
   - (tuỳ GitLab version, có thể cần thêm `read_api`, `write_repository` nếu yêu cầu)
4. Lưu lại token, dùng cho CI.

### 2.3. Tạo repo quản lý hạ tầng GitLab

Tạo một project mới, ví dụ:

- `gitlab-admin/infra-gitlab-config`

Clone repo về và tạo cấu trúc:

```bash
infra-gitlab-config/
├─ config/
│  └─ organizations.yml
├─ scripts/
│  └─ manage_gitlab.py
└─ .gitlab-ci.yml
```
---
## 3. Cấu hình tổ chức: config/organizations.yml
File này mô tả toàn bộ phòng ban, root group, sub-group, admin, members, projects.
Lưu ý: Thay toàn bộ *_example.com bằng email thật, username thật theo chuẩn nội bộ.
```bash
departments:
  cloudops:
    root_group: "cloudops"
    team_lead:
      username: "cloudops_tl"
      name: "CloudOps Team Lead"
      email: "cloudops_tl@example.com"
    subgroups:
      migrations:
        admin:
          username: "cloudops_mig_admin"
          name: "CloudOps Migrations Admin"
          email: "cloudops_mig_admin@example.com"
        members:
          - username: "cloudops_user1"
            name: "CloudOps User 1"
            email: "cloudops_user1@example.com"
            role: developer
        projects:
          - name: "mig-project-1"
          - name: "mig-project-2"

      operations:
        admin:
          username: "cloudops_ops_admin"
          name: "CloudOps Ops Admin"
          email: "cloudops_ops_admin@example.com"
        members:
          - username: "cloudops_user2"
            name: "CloudOps User 2"
            email: "cloudops_user2@example.com"
            role: developer
        projects:
          - name: "ops-monitoring"

      automations:
        admin:
          username: "cloudops_auto_admin"
          name: "CloudOps Auto Admin"
          email: "cloudops_auto_admin@example.com"
        members: []
        projects: []

      learning-testing:
        admin:
          username: "cloudops_learn_admin"
          name: "CloudOps Learn Admin"
          email: "cloudops_learn_admin@example.com"
        members: []
        projects: []

  devops:
    root_group: "devops"
    team_lead:
      username: "devops_tl"
      name: "DevOps Team Lead"
      email: "devops_tl@example.com"
    subgroups:
      platform-devsecops-tools:
        admin:
          username: "devops_platform_admin"
          name: "DevOps Platform Admin"
          email: "devops_platform_admin@example.com"
        members: []
        projects: []

      operations:
        admin:
          username: "devops_ops_admin"
          name: "DevOps Ops Admin"
          email: "devops_ops_admin@example.com"
        members: []
        projects: []

      ci-cd-projects:
        admin:
          username: "devops_ci_admin"
          name: "DevOps CI Admin"
          email: "devops_ci_admin@example.com"
        members:
          - username: "devops_dev1"
            name: "DevOps Dev 1"
            email: "devops_dev1@example.com"
            role: developer
        projects:
          - name: "cicd-template"
          - name: "project-a"
          - name: "project-b"

      learning-testing:
        admin:
          username: "devops_learn_admin"
          name: "DevOps Learn Admin"
          email: "devops_learn_admin@example.com"
        members: []
        projects: []

  appops:
    root_group: "appops"
    team_lead:
      username: "appops_tl"
      name: "AppOps Team Lead"
      email: "appops_tl@example.com"
    subgroups:
      applications-backend:
        admin:
          username: "appops_backend_admin"
          name: "AppOps Backend Admin"
          email: "appops_backend_admin@example.com"
        members: []
        projects: []

      operations:
        admin:
          username: "appops_ops_admin"
          name: "AppOps Ops Admin"
          email: "appops_ops_admin@example.com"
        members: []
        projects: []

      automations:
        admin:
          username: "appops_auto_admin"
          name: "AppOps Auto Admin"
          email: "appops_auto_admin@example.com"
        members: []
        projects: []

      learning-testing:
        admin:
          username: "appops_learn_admin"
          name: "AppOps Learn Admin"
          email: "appops_learn_admin@example.com"
        members: []
        projects: []

  developer:
    root_group: "developer"
    team_lead:
      username: "dev_tl"
      name: "Developer Team Lead"
      email: "dev_tl@example.com"
    subgroups:
      applications-dev-projects:
        admin:
          username: "dev_app_admin"
          name: "Dev Applications Admin"
          email: "dev_app_admin@example.com"
        members: []
        projects: []

      automations:
        admin:
          username: "dev_auto_admin"
          name: "Dev Auto Admin"
          email: "dev_auto_admin@example.com"
        members: []
        projects: []

      learning-testing:
        admin:
          username: "dev_learn_admin"
          name: "Dev Learn Admin"
          email: "dev_learn_admin@example.com"
        members: []
        projects: []

  tester:
    root_group: "tester"
    team_lead:
      username: "tester_tl"
      name: "Tester Team Lead"
      email: "tester_tl@example.com"
    subgroups:
      test-applications-projects:
        admin:
          username: "tester_app_admin"
          name: "Tester App Admin"
          email: "tester_app_admin@example.com"
        members: []
        projects: []

      automations:
        admin:
          username: "tester_auto_admin"
          name: "Tester Auto Admin"
          email: "tester_auto_admin@example.com"
        members: []
        projects: []

      learning-testing:
        admin:
          username: "tester_learn_admin"
          name: "Tester Learn Admin"
          email: "tester_learn_admin@example.com"
        members: []
        projects: []

  db:
    root_group: "db"
    team_lead:
      username: "db_tl"
      name: "DB Team Lead"
      email: "db_tl@example.com"
    subgroups:
      DB-applications-projects:
        admin:
          username: "db_app_admin"
          name: "DB App Admin"
          email: "db_app_admin@example.com"
        members: []
        projects: []

      automations:
        admin:
          username: "db_auto_admin"
          name: "DB Auto Admin"
          email: "db_auto_admin@example.com"
        members: []
        projects: []

      learning-testing:
        admin:
          username: "db_learn_admin"
          name: "DB Learn Admin"
          email: "db_learn_admin@example.com"
        members: []
        projects: []

```
