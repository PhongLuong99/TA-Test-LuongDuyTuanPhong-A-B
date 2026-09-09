# Technical Breakdown - Interactive Spell Casting System (TestA)

## 1. Implementation Approach
### 1.1 Cách tiếp cận

Tôi chia vòng đời của phép thành ba Niagara System có thể tái sử dụng, thay vì xây dựng toàn bộ hiệu ứng trong một system lớn:

- `NS_SpellCharge` xử lý giai đoạn báo hiệu và tích tụ năng lượng.
- `NS_SpellCast` thể hiện projectile đang bay cùng trail.
- `NS_SpellImpact` xử lý vụ nổ và hiệu ứng sau va chạm.

Trách nhiệm gameplay được phân chia giữa hai Blueprint:

- `BP_SpellCaster` quản lý input, trạng thái tụ lực, thời điểm kích hoạt và việc spawn projectile.
- `BP_Projectile` quản lý chuyển động, va chạm, vị trí impact, phản hồi vật lý, camera shake và dọn dẹp actor.

Cách tổ chức này cho phép preview và tinh chỉnh riêng từng giai đoạn. Niagara tập trung vào phần nhìn, còn Blueprint chịu trách nhiệm về quy tắc gameplay và va chạm.

### 1.2 Lý do lựa chọn giải pháp

- **Tính mô-đun:** charge, projectile và impact có thể được thay thế hoặc tái sử dụng độc lập.
- **Dễ lặp và tinh chỉnh:** Niagara asset, biến Blueprint, material parameter và thời gian trong Sequencer có thể được điều chỉnh mà không phải sửa toàn bộ hệ thống.
- **Phân chia trách nhiệm rõ ràng:** caster quyết định khi nào phép bắt đầu; projectile quyết định điều gì xảy ra sau khi được spawn.
- **Hiệu năng:** các particle mang tính hình ảnh sử dụng GPU simulation, trong khi va chạm ảnh hưởng gameplay được xử lý bằng hitbox trong Blueprint.
- **Khả năng trình bày:** gameplay thời gian thực và cinematic showcase dùng chung các asset cốt lõi.

## 2. UE-specific Workflows

### 2.1 Niagara System Creation

#### `NS_SpellCharge`

Hiệu ứng charge được ghép từ bốn GPU emitter, trong đó mỗi emitter đảm nhiệm một lớp hình ảnh:

| Emitter | Module chính | Renderer | Vai trò |
|---|---|---|---|
| `IceRock` | Burst, xoay mesh, color, scale mesh theo lifetime, dynamic material parameter | Mesh | Các mảnh rắn xoay quanh điểm tụ lực |
| `Aura` | Sphere Location, Gravity, Drag, Scale Color | Sprite | Lớp năng lượng mềm bao quanh |
| `Fountain` | Spawn Rate, Sphere Location, Add Velocity, Point Attraction Force, Scale Sprite Size | Sprite | Kéo particle hội tụ về tâm charge |
| `Fountain001` | Spawn Rate, Sphere Location, Sub-UV Animation, Point Attraction Force | Sprite | Lớp năng lượng động bổ sung |

`Point Attraction Force` hỗ trợ cảm giác năng lượng bị hút vào tâm. Kích thước và màu sắc được thay đổi theo tuổi thọ particle để quá trình bắt đầu và kết thúc charge dễ nhận biết.

#### `NS_SpellCast`

Hiệu ứng cast gồm hai GPU emitter:

- `DirectionalBurst` dùng Mesh Renderer để tạo phần lõi projectile.
- `Emit_ProjectileTrail` dùng Ribbon Renderer để tạo vệt chuyển động.

Ribbon sử dụng `Curl Noise Force`, `Scale Ribbon Width`, `Particles.RibbonTwist`, color và dynamic material parameter. Cách kết hợp này giúp trail bớt thẳng và cứng, đồng thời thu nhỏ và xoắn dần theo lifetime. Material `M_Trail` sử dụng hai lớp noise chuyển động bằng Panner cùng một nhánh dissolve điều khiển bằng parameter; `Particle Color` cung cấp màu và cường độ riêng cho particle.

#### `NS_SpellImpact`

Hiệu ứng impact sử dụng hai burst emitter:

- Emitter dùng Mesh Renderer bắn các mảnh theo `Add Velocity in Cone`, sau đó áp dụng Gravity và Drag.
- Emitter dùng Sprite Renderer phân bố particle từ hình cầu và chạy Sub-UV Animation dạng Linear cho lớp nổ hoặc khói thứ cấp.

System được `BP_Projectile` kích hoạt tại vị trí va chạm. Module Collision trong Niagara mesh emitter đang bị tắt ở cấu hình được ghi lại; vì vậy va chạm gameplay được xử lý bởi Blueprint hitbox thay vì Niagara collision event.

### 2.2 GPU vs CPU Simulation

Các Niagara emitter hiển thị trong project sử dụng GPU simulation vì đây là các tác vụ hình ảnh có thể chạy song song, gồm nhiều sprite, mesh particle và ribbon. GPU phù hợp với mật độ hiệu ứng cao, nhưng không nên trực tiếp quyết định gameplay vì việc đọc ngược collision hoặc event từ GPU bị hạn chế và có thể phát sinh thêm chi phí.

Do đó, project dùng hitbox phía CPU trong Blueprint cùng `OnComponentHit` làm sự kiện impact chính. Kết quả va chạm sau đó điều khiển vị trí Niagara, phản hồi camera và physics impulse.

### 2.3 Blueprint and Enhanced Input Workflow

`BP_SpellCaster` thêm `IMC_Default` thông qua Enhanced Input Local Player Subsystem và nhận sự kiện từ `IA_Interact`. Biến Boolean `IsSpellCharge` ngăn một lần cast mới bắt đầu khi chuỗi charge/cast hiện tại chưa kết thúc. Niagara Component của charge được tái sử dụng bằng `Set Active`; trạng thái khóa được giải phóng thông qua Niagara `OnSystemFinished`.

Luồng được xây dựng theo hướng event-driven và không cần dùng gameplay `Event Tick` cho thao tác cast và impact đã trình bày. Điều này tránh các phép tính Blueprint không cần thiết trên từng frame.

### 2.4 Collision and Feedback Workflow

`BP_Projectile` gồm các component:

- `Hitbox`
- `Arrow`
- Niagara Component `SpellCast`
- Niagara Component `SpellImpact`
- `ProjectileMovement`

Khi va chạm, Blueprint tắt trail, bật impact, đặt impact tại `Hit Result.Location`, vô hiệu hóa hitbox để tránh callback trùng, chạy client camera shake, chờ phần hậu cảnh của hiệu ứng hoàn tất rồi hủy projectile actor. Một nhánh hit khác kiểm tra `Is Simulating Physics` trước khi gọi `Add Impulse at Location` lên component bị trúng.

### 2.5 Level Sequencer Workflow

Project có master sequence `Final` và sáu take phục vụ trình bày:

- `Take1_Intro`
- `Take2_SpellReady`
- `Take3_SpellCharge1`
- `Take4_hit`
- `Take5_Fly`
- `Take6_End`

Việc chia showcase thành nhiều shot giúp dễ chỉnh sửa chuyển động camera và giữ master timeline gọn gàng. Camera giới thiệu giai đoạn charge, theo projectile và kết thúc tại impact. Kết quả được xuất thành MP4 để review.

### 2.6 Performance Validation

Ảnh profiling được cung cấp ghi nhận khoảng **60,01 FPS ở độ phân giải 1920×1080**, với **16,67 ms/frame**, **6,21 ms game thread**, **4,43 ms draw thread** và **5,36 ms GPU time** trong cảnh được chụp. README của project báo cáo khoảng **200 active particle**, thấp hơn giới hạn 500 particle trong đề bài. Đây là số liệu từ một ảnh chụp trong Editor, không phải benchmark đầy đủ trên nhiều cấu hình phần cứng.

## 3. Code / Blueprint Architecture


| `IA_Interact` / `IMC_Default` | Nhận input từ người chơi | `BP_SpellCaster` |
| `BP_SpellCaster` | Trạng thái cast, kích hoạt charge, timing, spawn projectile | `NS_SpellCharge`, `BP_Projectile` |
| `BP_Projectile` | Di chuyển, phát hiện hit, chuyển VFX, impulse, camera shake, cleanup | `NS_SpellCast`, `NS_SpellImpact`, hit component, Player Controller |
| `NS_SpellCharge` | VFX báo hiệu và tụ lực | Niagara Component trên caster |
| `NS_SpellCast` | Lõi projectile và ribbon trail | Niagara Component trên projectile |
| `NS_SpellImpact` | Vụ nổ và hiệu ứng sau va chạm | Niagara Component trên projectile |
| `M_Trail` | Noise chuyển động, màu, opacity/dissolve | Ribbon Renderer và Niagara dynamic parameter |
| `Final` cùng các shot sequence | Timing camera và cinematic presentation | Level actor, camera và VFX |

## 4. Problem-solving Process

### Tối ưu hiệu năng

**Thách thức:** Bốn charge emitter cùng mesh projectile, ribbon và impact burst có thể làm tăng particle cost và overdraw.

**Giải pháp:** Các emitter mang tính hình ảnh sử dụng GPU simulation, số lượng spawn và lifetime được giới hạn, projectile/impact actor được dọn dẹp rõ ràng. Ảnh profiling cho thấy cảnh test giữ mục tiêu 60 FPS; số active particle được báo cáo vẫn thấp hơn 500. Material Function Ribbon cho Trail


## 5. Asset Pipeline

1. Import texture, mesh và animation nguồn vào các thư mục content tương ứng.
2. Kiểm tra texture compression, alpha, tỷ lệ mesh, pivot, normal, skeleton assignment và animation retargeting.
3. Xây dựng material có thể tái sử dụng trước, bao gồm `M_Trail` và noise texture.
4. Tạo và preview độc lập từng Niagara System trong Niagara Editor.
5. Chỉ expose các parameter cần dùng bởi Blueprint hoặc Sequencer và đặt giá trị mặc định an toàn.
6. Tích hợp Niagara System vào `BP_SpellCaster` và `BP_Projectile` bằng component hoặc asset reference.
7. Kiểm tra collision, effect bounds, particle lifetime, overdraw và actor cleanup trong level `TestA-CombatVFX`.
8. Dựng các shot trong Level Sequencer và xuất MP4 cuối cùng.
