# Technical Breakdown - Interactive Spell Casting System (TestA)


## 1. Niagara Systems (as-built)

### 1.1. NS_SpellCharge — 4 Emitters

| **IceRock** | GPU | Emitter State (Self, Once), Spawn Burst Instantaneous | Initialize Particle | Particle State, Update Mesh Orientation (Rotation Rate), Color, Scale Mesh Size, Solve Forces and Velocity, Scale Color, Dynamic Material Parameters | **Mesh Renderer** |
| **Aura** | GPU | Emitter State (Self, Once), Spawn Burst Instantaneous | Initialize Particle, Shape Location (Sphere) | Particle State, Gravity Force, Drag, Solve Forces and Velocity, Scale Color, Dynamic Material Parameters | **Sprite Renderer** |
| **Fountain** | GPU | Emitter State (Self, Once), Spawn Rate | Initialize Particle, Shape Location (Sphere), Add Velocity (Linear) | Particle State, Scale Sprite Size, Drag, Scale Color, **Point Attraction Force**, Solve Forces and Velocity | **Sprite Renderer** |
| **Fountain001** | GPU | Emitter State (Self, Once), Spawn Rate | Initialize Particle, Shape Location (Sphere), Sub UV Animation (Random) | Particle State, Drag, Scale Color, Point Attraction Force, Solve Forces and Velocity | **Sprite Renderer** |

**Nhận xét kỹ thuật:**
- Kết hợp 4 emitter tạo lớp hiệu ứng charge nhiều tầng: `IceRock` (mesh xoay quanh tay, dùng Rotation Rate), `Aura` (hào quang sprite theo Gravity/Drag), `Fountain` + `Fountain001` (particle hút vào tâm bằng **Point Attraction Force**, đúng nguyên lý "tích năng lượng")
- Tất cả emitter dùng **GPU Sim** để tối ưu vì tổng particle count từ 4 emitter cộng dồn khá lớn
- `Fountain001` dùng Sub UV Animation Random để tạo biến thể hình ảnh cho particle giống tia lửa/tinh thể

---

### 1.2. NS_SpellCast — 2 Emitters

| **DirectionalBurst** | GPU | Emitter State (Self, Once), Spawn Burst Instantaneous | Initialize Particle | Particle State (active), Update Mesh Orientation (Rotation Rate), Solve Forces and Velocity | **Mesh Renderer** |
| **Emit_ProjectileTrail** | GPU | Emitter State (Self, Once), Spawn Rate | Initialize Particle | Particle State, **Curl Noise Force**, Color, Solve Forces and Velocity, Scale Ribbon Width, Set Particles.RibbonTwist, Dynamic Material Parameters | **Ribbon Renderer** |

**Nhận xét kỹ thuật:**
- `DirectionalBurst` (Mesh Renderer) đóng vai trò là "đầu đạn" solid bay theo hướng bắn
- `Emit_ProjectileTrail` (Ribbon Renderer) là đuôi vệt, dùng Material Function M_Trail , Dùng Panner trong Material để tạo chuyển động texture ,Curl Noise Force để khếch đại chuyển động xoáy tự nhiên cho trail thay vì đường thẳng cứng, thực hiện ở tầng Particle Update .
- `Scale Ribbon Width` + `RibbonTwist` (Dynamic Parameter) cho phép trail thon dần và xoắn theo thời gian sống

---

### 1.3. NS_SpellImpact — 2 Emitters

| **DirectionalBurst** | GPU | Emitter State (Self, Once), Spawn Burst Instantaneous | Initialize Particle, Add Velocity (In Cone) | Particle State, Gravity Force, Drag, Collision *(module tồn tại nhưng đang disable — icon mắt gạch chéo)*, Solve Forces and Velocity, Scale Color | **Mesh Renderer** |
| **DirectionalBurst001** | GPU | Emitter State (Self, Once), Spawn Burst Instantaneous | Initialize Particle, Shape Location (Sphere), Add Velocity (In Cone) | Particle State, Drag, Solve Forces and Velocity, Scale Color, Sub UV Animation (Linear) | **Sprite Renderer** |

**Nhận xét kỹ thuật:**
- Cả 2 emitter dùng **Add Velocity In Cone** → tạo hình nón văng mảnh vỡ/hạt sáng ra từ điểm va chạm, đúng đặc trưng vụ nổ
- Các mesh renderer có set collision với lifetime 2s
- `DirectionalBurst001` dùng Sub UV Animation Linear để chạy texture animation Smoke theo trình tự cố định, tạo hiệu ứng khói từ vụ nổ

---

## 2. Blueprint Integration

### 2.1. Input Setup & Spawn Flow (BP_SpellCaster)

Flow khi nhấn phím (Enhanced Input Action → IA_Interact, Triggered)
- Dùng biến bool `Is Spell Charge` làm cờ chặn spam input — người chơi không thể bắn liên tục khi đang charge/cast
- Việc **Set Active** thay vì Spawn System at Location cho phép tái sử dụng cùng 1 Niagara Component đã gắn sẵn trên actor, tiết kiệm chi phí khởi tạo
- 2 lệnh "Spawn Spell" với 2 mốc Delay (3.5s charge + 1.2s) khớp với timeline: charge → release → travel trước khi hệ thống tự dọn dẹp qua `OnSystemFinished`của Niagara của SpellCharge

---

### 2.2. Add Physics Impulse (BP_Projectile — Event Hit)

- Chỉ áp impulse khi vật thể bị trúng đang **Simulating Physics** (tránh lỗi gọi impulse lên actor static/không có physics)
- Impulse hướng theo vector vận tốc hiện tại của đạn tại thời điểm va chạm — tạo cảm giác "đẩy" vật thể theo đúng hướng bắn thay vì một hướng cố định
- `Destroy Actor` dọn dẹp projectile ngay sau khi impulse được áp — tránh actor tồn đọng trong scene

---

### 2.3. Hitbox Collision → Trigger Impact & Explode (BP_Projectile — On Component Hit)
- Thứ tự xử lý: **tắt trail → bật impact tại đúng vị trí va chạm → khóa collision hitbox → rung camera** — đảm bảo hiệu ứng nổ chỉ trigger đúng 1 lần
- `Set World Location` dùng chính Hit Location từ Break Hit Result, không hardcode tọa độ — impact luôn xuất hiện đúng điểm chạm bất kể vị trí target
- `Set Collision Enabled → No Collision` là kỹ thuật quan trọng để chặn `On Component Hit` bị gọi lại nhiều lần trong cùng 1 frame va chạm (double-trigger bug)
- Camera Shake dùng biến `Scale Camera Hit` — cho phép Designer tinh chỉnh cường độ rung theo từng loại spell mà không sửa Blueprint
- `Delay 4.0s` sau cùng dùng để giữ actor tồn tại đủ lâu cho NS_SpellImpact chạy hết vòng đời rồi mới cleanup (Destroy Actor, không thấy trong đoạn graph nhưng là bước logic tiếp theo)

---

## 4. Level Sequencer 

| Camera Rig | Dolly quanh nhân vật, focus vào tay đang charge → pan theo hướng đạn bay → cắt cảnh cận impact 
| Export | Resolution 1920x1080, 30fps, format .mp4 (H.264) 