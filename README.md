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
---
## 3.2. Repo hạ tầng – gitlab-admin/infra-gitlab-config
cấu trúc:
```bash
infra-gitlab-config/
├─ config/
│  ├─ base-cloudops.yml      # cấu trúc "cứng" do TL/DevOps quản lý (optional)
│  ├─ base-devops.yml
│  ├─ base-appops.yml
│  ├─ base-developer.yml
│  ├─ base-tester.yml
│  ├─ base-db.yml
│  ├─ cloudops.yml           # sync từ repo phòng CloudOps
│  ├─ devops.yml
│  ├─ appops.yml
│  ├─ developer.yml
│  ├─ tester.yml
│  └─ db.yml
├─ scripts/
│  └─ manage_gitlab.py
└─ .gitlab-ci.yml

```
### 3.2.1. scripts/manage_gitlab.py
  - Hỗ trợ đọc nhiều file config/*.yml (load_all_departments).
  - Có validation nâng cao (trùng username/email, conflict với GitLab).
  - Tạo user + group + subgroup + project + phân quyền.
  - Có comment chỗ để chuyển sang chế độ LDAP-only (không tự tạo user).
```bash
import os
import sys
import glob
import requests
import yaml
from urllib.parse import quote

# ============================================
#  CONFIG & CONSTANTS
# ============================================

GITLAB_URL = os.environ["GITLAB_URL"]          # vd: https://gitlab.example.com
API_TOKEN = os.environ["GITLAB_API_TOKEN"]     # admin token

HEADERS = {
    "PRIVATE-TOKEN": API_TOKEN,
    "Content-Type": "application/json"
}

ROLE_MAP = {
    "owner": 50,
    "maintainer": 40,
    "developer": 30,
    "reporter": 20,
    "guest": 10
}

# ============================================
#  LOW-LEVEL API HELPERS
# ============================================

def gitlab_get(path, params=None):
    r = requests.get(f"{GITLAB_URL}/api/v4{path}", headers=HEADERS, params=params)
    if r.status_code == 404:
        return None
    r.raise_for_status()
    return r.json()

def gitlab_post(path, data):
    r = requests.post(f"{GITLAB_URL}/api/v4{path}", headers=HEADERS, json=data)
    r.raise_for_status()
    return r.json()

def gitlab_put(path, data):
    r = requests.put(f"{GITLAB_URL}/api/v4{path}", headers=HEADERS, json=data)
    r.raise_for_status()
    return r.json()

# ============================================
#  USER FUNCTIONS
# ============================================

def find_user_by_username(username):
    if not username:
        return None
    users = gitlab_get("/users", params={"username": username})
    return users[0] if users else None

def find_users_by_email(email):
    """
    Tìm user theo email. Tùy version GitLab, có thể không hỗ trợ filter trực tiếp,
    nên ở đây dùng search rồi lọc lại.
    """
    if not email:
        return []
    users = gitlab_get("/users", params={"search": email}) or []
    return [u for u in users if (u.get("email") or "").lower() == email.lower()]

def create_user_if_not_exists(username, name, email):
    """
    Chế độ LOCAL USER:
      - Nếu user chưa tồn tại trong GitLab thì tạo mới.
      - Dùng cho môi trường không/hoặc không chỉ dùng LDAP.

    Nếu muốn chuyển sang chế độ LDAP-only (KHÔNG tạo user),
    đổi hàm này thành chỉ find_user_by_username và return user (hoặc None).

    Ví dụ cho LDAP-only:
        def create_user_if_not_exists(username, name, email):
            if not username:
                return None
            return find_user_by_username(username)
    """
    if not username or not email:
        return None
    user = find_user_by_username(username)
    if user:
        return user

    data = {
        "username": username,
        "name": name or username,
        "email": email,
        "reset_password": True,
        "skip_confirmation": True
    }
    return gitlab_post("/users", data)

# ============================================
#  GROUP FUNCTIONS
# ============================================

def find_group_by_full_path(full_path):
    if not full_path:
        return None
    # /groups/:id chấp nhận full_path
    group = gitlab_get(f"/groups/{full_path}")
    return group

def create_group(name, path, parent_id=None):
    data = {"name": name, "path": path}
    if parent_id:
        data["parent_id"] = parent_id
    return gitlab_post("/groups", data)

def ensure_root_group(dept_key, root_group_path):
    group = find_group_by_full_path(root_group_path)
    if group:
        return group
    return create_group(dept_key, root_group_path)

def ensure_subgroup(parent_group, subgroup_name):
    full_path = f"{parent_group['full_path']}/{subgroup_name}"
    group = find_group_by_full_path(full_path)
    if group:
        return group
    return create_group(subgroup_name, subgroup_name, parent_group["id"])

def add_member_to_group(group_id, user_id, access_level):
    """
    Thêm member vào group. Nếu đã tồn tại thì update access_level.
    """
    try:
        gitlab_post(f"/groups/{group_id}/members", {
            "user_id": user_id,
            "access_level": access_level
        })
    except requests.exceptions.HTTPError as e:
        if e.response.status_code == 409:
            gitlab_put(f"/groups/{group_id}/members/{user_id}", {
                "access_level": access_level
            })
        else:
            raise

# ============================================
#  PROJECT FUNCTIONS
# ============================================

def find_project_by_full_path(full_path):
    encoded = quote(full_path, safe='')
    project = gitlab_get(f"/projects/{encoded}")
    return project

def ensure_project(group, project_name):
    full_path = f"{group['full_path']}/{project_name}"
    project = find_project_by_full_path(full_path)
    if project:
        return project
    data = {
        "name": project_name,
        "path": project_name,
        "namespace_id": group["id"],
        "visibility": "private"
    }
    return gitlab_post("/projects", data)

# ============================================
#  CONFIG LOADER (MULTI-FILE)
# ============================================

def load_all_departments():
    """
    Đọc tất cả file YAML trong thư mục config/*.yml
    và merge vào 1 cấu trúc config chung:
      {
        "departments": {
          "cloudops": {...},
          "devops": {...},
          ...
        }
      }
    Nếu dept trùng tên ở nhiều file, file đọc sau sẽ override/ghi đè.
    """
    data = {"departments": {}}
    for path in glob.glob("config/*.yml"):
        with open(path) as f:
            part = yaml.safe_load(f) or {}
            for dept_key, dept_conf in (part.get("departments") or {}).items():
                if dept_key not in data["departments"]:
                    data["departments"][dept_key] = dept_conf
                else:
                    # Ghi đè đơn giản; nếu muốn merge sâu hơn cần viết thêm logic
                    data["departments"][dept_key].update(dept_conf)
    return data

# ============================================
#  VALIDATION – COLLECT USERS
# ============================================

def collect_users_from_config(config):
    """
    Thu thập toàn bộ user được khai báo trong YAML, gồm:
      - team_lead
      - admin của từng subgroup
      - members của từng subgroup

    Trả về list dict:
      {
        "username": ...,
        "email": ...,
        "name": ...,
        "source": "cloudops.team_lead" / "devops.ci-cd-projects.member" ...
      }
    """
    users = []

    for dept_key, dept_conf in config.get("departments", {}).items():
        # team_lead
        tl = dept_conf.get("team_lead")
        if tl:
            users.append({
                "username": tl.get("username"),
                "email": tl.get("email"),
                "name": tl.get("name"),
                "source": f"{dept_key}.team_lead"
            })

        # subgroups
        for sg_name, sg_conf in (dept_conf.get("subgroups") or {}).items():
            admin = sg_conf.get("admin")
            if admin:
                users.append({
                    "username": admin.get("username"),
                    "email": admin.get("email"),
                    "name": admin.get("name"),
                    "source": f"{dept_key}.{sg_name}.admin"
                })
            for m in sg_conf.get("members", []):
                users.append({
                    "username": m.get("username"),
                    "email": m.get("email"),
                    "name": m.get("name"),
                    "source": f"{dept_key}.{sg_name}.member"
                })

    return users

# ============================================
#  VALIDATION – DUPLICATES TRONG YAML
# ============================================

def validate_local_duplicates(users):
    """
    Kiểm tra trùng username / email trong chính file YAML (tất cả dept).
    """
    errors = []
    username_map = {}
    email_map = {}

    for u in users:
        uname = (u.get("username") or "").strip()
        email = (u.get("email") or "").strip().lower()
        src = u.get("source")

        if uname:
            if uname in username_map and username_map[uname] != src:
                errors.append(
                    f"Username '{uname}' được khai báo ở nhiều nơi: "
                    f"{username_map[uname]} và {src}"
                )
            else:
                username_map[uname] = src

        if email:
            if email in email_map and email_map[email] != src:
                errors.append(
                    f"Email '{email}' được khai báo ở nhiều nơi: "
                    f"{email_map[email]} và {src}"
                )
            else:
                email_map[email] = src

    return errors

# ============================================
#  VALIDATION – CONFLICT VỚI GITLAB
# ============================================

def validate_with_gitlab(users):
    """
    Kiểm tra xung đột với GitLab:
      - username đã tồn tại nhưng email khác
      - email đã tồn tại nhưng username khác
    """
    errors = []

    usernames = sorted({(u.get("username") or "").strip() for u in users if u.get("username")})
    emails = sorted({(u.get("email") or "").strip().lower() for u in users if u.get("email")})

    yaml_by_username = {
        u["username"]: {
            "email": (u.get("email") or "").strip().lower(),
            "name": u.get("name"),
            "source": u.get("source")
        }
        for u in users if u.get("username")
    }
    yaml_by_email = {
        (u.get("email") or "").strip().lower(): {
            "username": u.get("username"),
            "name": u.get("name"),
            "source": u.get("source")
        }
        for u in users if u.get("email")
    }

    # Check theo username
    for uname in usernames:
        gl_user = find_user_by_username(uname)
        if not gl_user:
            continue
        yaml_u = yaml_by_username[uname]
        yaml_email = yaml_u["email"]
        gl_email = (gl_user.get("email") or "").strip().lower()
        if yaml_email and gl_email and yaml_email != gl_email:
            errors.append(
                f"CONFLICT: Username '{uname}' trong GitLab đã tồn tại với email '{gl_email}', "
                f"nhưng YAML khai báo email '{yaml_email}' (source: {yaml_u['source']})."
            )

    # Check theo email
    for email in emails:
        gl_users = find_users_by_email(email)
        if not gl_users:
            continue
        yaml_u = yaml_by_email[email]
        yaml_uname = (yaml_u.get("username") or "").strip()
        for gl_user in gl_users:
            gl_uname = (gl_user.get("username") or "").strip()
            if yaml_uname and gl_uname and yaml_uname != gl_uname:
                errors.append(
                    f"CONFLICT: Email '{email}' trong GitLab đã gắn với username '{gl_uname}', "
                    f"nhưng YAML khai báo username '{yaml_uname}' (source: {yaml_u['source']})."
                )

    return errors

# ============================================
#  OPTIONAL – VALIDATION CHO LDAP-ONLY
# ============================================

def validate_ldap_users_exist_in_gitlab(users):
    """
    Dùng nếu môi trường LDAP-only, KHÔNG cho phép tạo user mới:
      - Kiểm tra tất cả username trong YAML đã tồn tại trên GitLab chưa.
      - Nếu chưa thì báo lỗi, yêu cầu user login GitLab ít nhất 1 lần trước.
    Nếu không dùng LDAP-only, có thể không gọi hàm này.
    """
    errors = []
    checked = set()

    for u in users:
        uname = (u.get("username") or "").strip()
        if not uname or uname in checked:
            continue
        checked.add(uname)
        gl_user = find_user_by_username(uname)
        if not gl_user:
            errors.append(
                f"LDAP user '{uname}' chưa tồn tại trong GitLab. "
                f"Yêu cầu user đăng nhập GitLab ít nhất 1 lần trước khi gán quyền. "
                f"(source: {u.get('source')})"
            )

    return errors

# ============================================
#  ENTRY VALIDATION
# ============================================

def run_advanced_validation(config, ldap_only=False):
    """
    Gồm:
      - Kiểm tra trùng username/email trong YAML.
      - Kiểm tra conflict với GitLab.
      - (Tùy chọn) Kiểm tra user LDAP đã có trong GitLab.
    """
    users = collect_users_from_config(config)
    errors = []

    errors.extend(validate_local_duplicates(users))
    errors.extend(validate_with_gitlab(users))

    if ldap_only:
        errors.extend(validate_ldap_users_exist_in_gitlab(users))

    if errors:
        print("=== VALIDATION ERRORS DETECTED ===")
        for e in errors:
            print(f"- {e}")
        print("=== END OF ERRORS ===")
        sys.exit(1)

# ============================================
#  APPLY CONFIG – PER DEPARTMENT
# ============================================

def process_department(dept_key, dept_conf):
    """
    Tạo/cập nhật:
      - Root group (theo dept_conf["root_group"], nếu có).
      - Subgroups.
      - Admin + members.
      - Projects.
    """
    root_path = dept_conf.get("root_group", dept_key)
    root_group = ensure_root_group(dept_key, root_path)

    # Team lead -> Owner của root group (nếu khai báo)
    tl = dept_conf.get("team_lead", {})
    if tl:
        tl_user = create_user_if_not_exists(
            tl.get("username"),
            tl.get("name"),
            tl.get("email")
        )
        if tl_user:
            add_member_to_group(root_group["id"], tl_user["id"], ROLE_MAP["owner"])

    for subgroup_name, subgroup_conf in (dept_conf.get("subgroups") or {}).items():
        sg = ensure_subgroup(root_group, subgroup_name)

        # Admin subgroup = Maintainer
        admin = subgroup_conf.get("admin", {})
        if admin:
            admin_user = create_user_if_not_exists(
                admin.get("username"),
                admin.get("name"),
                admin.get("email")
            )
            if admin_user:
                add_member_to_group(sg["id"], admin_user["id"], ROLE_MAP["maintainer"])

        # Members
        for member in subgroup_conf.get("members", []):
            u = create_user_if_not_exists(
                member.get("username"),
                member.get("name") or member.get("username"),
                member.get("email")
            )
            if not u:
                continue
            role = member.get("role", "developer")
            access_level = ROLE_MAP.get(role, ROLE_MAP["developer"])
            add_member_to_group(sg["id"], u["id"], access_level)

        # Projects
        for proj in subgroup_conf.get("projects", []):
            ensure_project(sg, proj["name"])

# ============================================
#  MAIN
# ============================================

def main():
    config = load_all_departments()

    # ldap_only = False: cho phép script tạo user mới
    # ldap_only = True: CHỈ dùng user đã tồn tại (LDAP-only)
    ldap_only = False

    run_advanced_validation(config, ldap_only=ldap_only)

    for dept_key, dept_conf in config["departments"].items():
        process_department(dept_key, dept_conf)

if __name__ == "__main__":
    main()

```
  - Validation nâng cao (run_advanced_validation(config)):
    - Thu thập tất cả user từ YAML.
    - Kiểm tra:
      - Trùng username / email trong YAML.
      - Conflict với user/email hiện có trên GitLab.
      - (Tùy chọn) LDAP user chưa tồn tại trên GitLab.
    - Nếu lỗi → in danh sách lỗi + sys.exit(1) → CI fail, không apply.
  - Apply (process_department):
    - Đảm bảo root group tồn tại (ensure_root_group).
    - Đảm bảo subgroups tồn tại (ensure_subgroup).
    - Tạo projects nếu chưa có (ensure_project).
    - Tạo user (nếu cho phép local user) hoặc chỉ find user (nếu dùng LDAP).
    - Gán quyền:
      - Team Lead: Owner trên root group.
      - Admin subgroup: Maintainer.
      - Members: Developer (hoặc role khác theo YAML).
### 3.2.2. .gitlab-ci.yml trong repo infra
