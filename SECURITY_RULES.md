# 🔒 Firebase Security Rules — Hướng dẫn bảo mật

## Về API Key

Firebase API key **không phải là secret** — nó là public identifier giống như URL.
Google thiết kế nó để dùng trong client-side code.

**Bảo mật thực sự nằm ở Firebase Security Rules** — quy tắc đọc/ghi dữ liệu
được kiểm tra phía server Firebase, không phải phía client.

## Cách thiết lập Security Rules

1. Vào [Firebase Console](https://console.firebase.google.com/)
2. Chọn project **binhson-confessions**
3. Vào **Realtime Database** → tab **Rules**
4. Thay nội dung bằng rules bên dưới

## Rules đề xuất

```json
{
  "rules": {
    "confessions": {
      // Ai cũng đọc được bài đã duyệt
      ".read": true,
      "$postId": {
        // Không cho phép ghi trực tiếp (chỉ admin duyệt từ pending)
        ".write": false,
        "likes": {
          // Cho phép cập nhật lượt thích
          ".write": true
        },
        "likedBy": {
          ".write": true
        },
        "comments": {
          // Cho phép thêm bình luận
          ".write": true
        },
        "views": {
          // Cho phép cập nhật lượt xem
          ".write": true
        }
      }
    },

    "pending_posts": {
      // Không cho phép đọc danh sách chờ duyệt
      ".read": false,
      // Cho phép người dùng TẠO bài mới (push)
      ".write": "newData.exists() && !data.exists()"
    },

    "reports": {
      // Không cho phép đọc báo cáo
      ".read": false,
      "$postId": {
        // Cho phép tạo báo cáo mới
        ".write": "newData.exists() && !data.exists()"
      }
    },

    "announcements": {
      // Ai cũng đọc được thông báo
      ".read": true,
      // Không cho phép ghi (chỉ admin)
      ".write": false
    },

    "online_users": {
      ".read": true,
      ".write": true
    },

    "daily_prompt": {
      ".read": true,
      ".write": false
    }
  }
}
```

## ⚠️ Lưu ý quan trọng

Với rules trên, trang **admin (`login.html`) sẽ không hoạt động** vì admin
cũng bị chặn đọc/ghi `pending_posts`, `reports`, và ghi `confessions`.

### Giải pháp cho admin:

**Cách 1 (Khuyến nghị): Dùng Firebase Authentication**
- Tạo tài khoản admin trên Firebase Console → Authentication → Users
- Thêm Firebase Auth vào `login.html`
- Cập nhật rules dùng `auth.uid`:

```json
{
  "rules": {
    "pending_posts": {
      ".read": "auth != null && auth.uid === 'YOUR_ADMIN_UID'",
      ".write": "(!data.exists() && newData.exists()) || (auth != null && auth.uid === 'YOUR_ADMIN_UID')"
    },
    "reports": {
      ".read": "auth != null && auth.uid === 'YOUR_ADMIN_UID'",
      ".write": true
    },
    "confessions": {
      ".read": true,
      "$postId": {
        ".write": "(auth != null && auth.uid === 'YOUR_ADMIN_UID') || (!data.exists() && newData.exists())",
        "likes": { ".write": true },
        "likedBy": { ".write": true },
        "comments": { ".write": true },
        "views": { ".write": true }
      }
    },
    "announcements": {
      ".read": true,
      ".write": "auth != null && auth.uid === 'YOUR_ADMIN_UID'"
    }
  }
}
```

**Cách 2 (Tạm thời): Giữ rules mở cho admin**
Nếu chưa muốn setup Authentication, ít nhất hãy thêm validation rules
để chặn dữ liệu rác:

```json
{
  "rules": {
    "pending_posts": {
      ".read": true,
      ".write": true,
      "$postId": {
        ".validate": "newData.hasChildren(['content', 'timestamp'])"
      }
    },
    "confessions": {
      ".read": true,
      ".write": true,
      "$postId": {
        ".validate": "newData.hasChildren(['content', 'timestamp'])"
      }
    }
  }
}
```

## Thêm bảo vệ API Key (tuỳ chọn)

Vào [Google Cloud Console](https://console.cloud.google.com/) → APIs & Services → Credentials:
1. Chọn API key của project
2. Thêm **Application restrictions**: HTTP referrers
3. Thêm domain website của bạn (VD: `yourdomain.com/*`)
4. Thêm **API restrictions**: chỉ cho phép Firebase Realtime Database API

→ Dù ai có key cũng không dùng được từ domain khác.
