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
