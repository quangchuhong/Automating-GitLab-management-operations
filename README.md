# GitLab IaC Architecture – Multi-Repo, Per-Department Input

Tài liệu này mô tả kiến trúc mới và hướng dẫn triển khai đầy đủ cho hệ thống:

- Mỗi phòng ban có **repo riêng** để tự khai báo user/project.
- Một **repo hạ tầng trung tâm** gom tất cả cấu hình, validate nâng cao và apply lên GitLab bằng API.

---

## 1. Mục tiêu

- Tạo và quản lý:
  - **Groups / Subgroups** theo từng phòng ban.
  - **Projects** trong từng subgroup.
  - **Users** và **quyền** (Owner, Maintainer, Developer).
- Cho phép **từng phòng tự chủ** khai báo thông tin của mình:
  - Không nhìn thấy thông tin các phòng khác.
- Tự động hoá:
  - Sync cấu hình từ repo phòng → repo hạ tầng.
  - Validate nâng cao (trùng username, email, conflict với GitLab).
  - Gọi GitLab API apply cấu hình.

---

## 2. Kiến trúc tổng quan

### 2.1. Các repository

1. **Repo hạ tầng trung tâm** (chỉ TL/DevOps xem được):

   - Ví dụ:  
     `gitlab-admin/infra-gitlab-config`

   - Chứa:
     - `config/*.yml`: cấu hình tổng hợp từ tất cả phòng.
     - `scripts/manage_gitlab.py`: script gọi GitLab API.
     - `.gitlab-ci.yml`: pipeline sync → validate → apply.

2. **Mỗi phòng một repo request riêng** (user phòng đó xem/sửa được):

   - Ví dụ:
     - `cloudops/gitlab-requests`
     - `devops/gitlab-requests`
     - `appops/gitlab-requests`
     - `developer/gitlab-requests`
     - `tester/gitlab-requests`
     - `db/gitlab-requests`

   - Mỗi repo chứa:
     - `config/<dept>.yml` – file YAML để phòng nhập thông tin.
     - `.gitlab-ci.yml` – chỉ validate YAML ở MR.
     - `README.md` – hướng dẫn cho user phòng đó.

### 2.2. Data flow

1. User phòng X sửa `config/<dept>.yml` trong repo `X/gitlab-requests`.
2. User tạo MR → CI của repo phòng X validate YAML → TL phòng X merge vào `main`.
3. Repo `infra-gitlab-config` (branch `main`) chạy pipeline:
   - Job `sync-configs`:
     - Dùng `curl` + token để tải các file YAML từ từng repo phòng (branch `main`) về `infra-gitlab-config/config/`.
   - Job `validate-config`:
     - Gọi `scripts/manage_gitlab.py` (hàm `load_all_departments + run_advanced_validation`) để validate:
       - Trùng username/email trong YAML.
       - Conflict với user/email đang có trong GitLab.
       - (Tuỳ chọn) Kiểm tra LDAP user đã tồn tại trên GitLab.
   - Job `apply-config`:
     - Gọi GitLab API để tạo/ cập nhật group, subgroup, project, members, role.

---

## 3. Cấu trúc chi tiết các repo

### 3.1. Repo phòng ban – ví dụ CloudOps

**Tên repo gợi ý**: `cloudops/gitlab-requests`

Cấu trúc:

```bash
cloudops-gitlab-requests/
├─ config/
│  └─ cloudops.yml
└─ .gitlab-ci.yml
└─ README.md
```
### 3.1.1. config/cloudops.yml (mẫu)
```bash
departments:
  cloudops:
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

```
  - Các repo phòng khác (DevOps, AppOps, Developer, Tester, DB) dùng cấu trúc tương tự, chỉ khác tên department & subgroups.

### 3.1.2. .gitlab-ci.yml trong repo phòng
Dùng chung cho tất cả phòng, chỉ đổi tên file YAML:
```bash
stages:
  - validate

validate-config:
  stage: validate
  image: python:3.11
  script:
    - pip install pyyaml
    - python -c "import yaml; yaml.safe_load(open('config/cloudops.yml'))"
  only:
    - merge_requests

```
  - Với DevOps: config/devops.yml
  - AppOps: config/appops.yml
  - Developer: config/developer.yml
  - Tester: config/tester.yml
  - DB: config/db.yml
