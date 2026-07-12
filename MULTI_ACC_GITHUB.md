# Quy trình làm việc với nhiều GitHub account (SSH)

## Setup ban đầu (đã làm 1 lần, không cần lặp lại)

1. Tạo 2 SSH key riêng:
```bash
ssh-keygen -t ed25519 -C "hphuc15" -f ~/.ssh/id_ed25519_hphuc15
ssh-keygen -t ed25519 -C "hphuc15v" -f ~/.ssh/id_ed25519_hphuc15v
```

2. Add từng public key (`.pub`) vào đúng account trên GitHub → Settings → SSH and GPG keys.

3. Khai báo alias trong `~/.ssh/config`:
```
Host github-hphuc15
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_hphuc15

Host github-hphuc15v
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_hphuc15v
```

4. Load key vào ssh-agent (Windows):
```powershell
Get-Service ssh-agent | Set-Service -StartupType Automatic
Start-Service ssh-agent
ssh-add ~/.ssh/id_ed25519_hphuc15
ssh-add ~/.ssh/id_ed25519_hphuc15v
ssh-add -l   # kiểm tra key đã load
```

5. Test kết nối:
```bash
ssh -T github-hphuc15
ssh -T github-hphuc15v
```
→ Kết quả mong đợi: `Hi <username>! You've successfully authenticated...`

**Lưu ý permission trên Windows:** nếu gặp lỗi `Bad permissions` / `UNPROTECTED PRIVATE KEY FILE`, cần giới hạn ACL của file key và file `config` chỉ cho user hiện tại + SYSTEM:
```powershell
icacls <file> /inheritance:r
icacls <file> /grant:r "TÊN-MÁY\username:F"
icacls <file> /grant:r "NT AUTHORITY\SYSTEM:F"
```
Nếu vẫn lỗi, kiểm tra owner của file (`Get-Acl <file> | Select Owner`) — phải đúng là user hiện tại, không phải `Administrator`. Nếu sai, mở PowerShell **as Administrator** rồi chạy `icacls <file> /setowner "TÊN-MÁY\username"`.

---

## Cơ chế: 3 việc độc lập nhau

| Việc | Quyết định bởi |
|---|---|
| Dùng key nào để xác thực (có quyền push hay không) | Host alias trong remote URL (`github-hphuc15` hay `github-hphuc15v`) |
| Commit đứng tên ai (author) | `git config user.name` / `user.email` — set riêng từng repo |
| Folder lưu ở đâu trên máy | Tùy ý, không liên quan 2 cái trên |

**Không có khái niệm "chuyển account trên máy".** Cả 2 account luôn sẵn sàng cùng lúc; mỗi **repo** tự gắn cố định với 1 account thông qua alias + config local.

SSH và HTTPS không loại trừ nhau — đổi qua lại bằng `git remote set-url` bất cứ lúc nào, không ảnh hưởng repo khác.

---

## A. Clone repo có sẵn (của người khác hoặc của mình)

```bash
git clone git@github-hphuc15v:owner/repo.git
cd repo
git config user.name "hphuc15v"
git config user.email "email-của-hphuc15v"
```

Sau đó dùng bình thường:
```bash
git add .
git commit -m "message"
git push
```

**Điều kiện để push được:** account `hphuc15v` phải có quyền trên repo (là chủ, hoặc được add làm collaborator / thuộc org có quyền). Nếu chưa, chủ repo cần vào Settings → Collaborators → Add people.

---

## B. Đổi account cho repo đã clone từ trước (VD từng dùng HTTPS hoặc account khác)

```bash
cd repo
git remote set-url origin git@github-hphuc15v:owner/repo.git
git config user.name "hphuc15v"
git config user.email "email-của-hphuc15v"
```

Nếu commit trước đó đã lỡ push sai author, sửa commit gần nhất:
```bash
git commit --amend --reset-author --no-edit
git push --force
```

---

## C. Tạo project mới ở local, push lên repo trống

1. Trên GitHub, đăng nhập đúng account, tạo repo trống (không tick "Add README" để tránh conflict).

2. Trong thư mục project local:
```bash
git init
git add .
git commit -m "initial commit"
git config user.name "hphuc15v"
git config user.email "email-của-hphuc15v"
git remote add origin git@github-hphuc15v:hphuc15v/tên-repo.git
git branch -M main
git push -u origin main
```

Nếu repo trên GitHub không hoàn toàn trống (có README), trước khi push:
```bash
git pull origin main --allow-unrelated-histories
```

---

## Kiểm tra nhanh 1 repo đang dùng account nào

```bash
git remote -v
git config user.name
git config user.email
```

---

## Ghi chú quan trọng

- **GitHub xác định author commit dựa vào `user.email`**, không phải SSH key dùng để push. Quên set email → commit vẫn hiện account cũ (global config).
- Email dùng trong `user.email` phải **verify trong Settings → Emails** của đúng account đó, nếu không GitHub sẽ không link commit đúng người.
- Muốn tự động set `user.name`/`user.email` theo thư mục workspace (không cần gõ tay mỗi repo mới), dùng `includeIf` trong `~/.gitconfig`:
```ini
[includeIf "gitdir:D:/WorkSpace/hphuc15/"]
    path = ~/.gitconfig-hphuc15
[includeIf "gitdir:D:/WorkSpace/hphuc15v/"]
    path = ~/.gitconfig-hphuc15v
```
(path phải dùng `/`, không dùng `\`, và có dấu `/` ở cuối)