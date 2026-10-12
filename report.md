# Session 11 - Bài 3: Tối ưu tốc độ CI bằng Gradle Dependency Caching

- Repository: <https://github.com/thang-260704/IT209_Ss11_Ex3>
- Run #1 (chưa có cache): <https://github.com/thang-260704/IT209_Ss11_Ex3/actions/runs/38195480048>
- Run #2 (đã có cache): <https://github.com/thang-260704/IT209_Ss11_Ex3/actions/runs/38195653920>

## 1. Cấu hình

Trong `.github/workflows/ci.yml`, bước `actions/setup-java@v5` bổ sung `cache: 'gradle'`:

```yaml
      - name: Thiết lập môi trường Java JDK 17
        uses: actions/setup-java@v5
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: 'gradle'
```

**Cơ chế hoạt động:**

1. **Đầu job (step setup-java):** tính khoá cache từ hệ điều hành và hash của các tệp `**/*.gradle*`, `**/gradle-wrapper.properties`, rồi tìm cache có khoá đó. Tìm thấy thì giải nén `~/.gradle/caches` và `~/.gradle/wrapper` vào runner.
2. **Cuối job (step `Post Thiết lập môi trường Java JDK 17`):** nếu lúc đầu không tìm thấy cache thì nén hai thư mục trên và lưu lên GitHub với khoá vừa tính.
3. Khoá chỉ đổi khi `build.gradle` hoặc phiên bản Gradle Wrapper thay đổi. Commit chỉ sửa code vẫn dùng lại được cache cũ.

Cache gắn với từng repository, nên lượt chạy đầu tiên của repo mới luôn là lượt "lạnh".

## 2. Cách đo

| Lượt | Commit | Mô tả |
|---|---|---|
| Run #1 | `34eb5e5` | Push lần 1 vào repo mới: chưa có cache, phải tải Gradle 8.x và toàn bộ dependency từ Internet |
| Run #2 | `fb3ec5b` | `git commit --allow-empty` rồi push lần 2: mã nguồn và `build.gradle` không đổi, khoá cache giống run #1 |

## 3. Kết quả so sánh

| Hạng mục | Run #1 (cold) | Run #2 (warm) | Chênh lệch |
|---|---:|---:|---:|
| Thiết lập môi trường Java JDK 17 | 1s | 6s | +5s (tải và giải nén cache) |
| **Thực thi JUnit Test với Gradle** | **49s** | **18s** | **−31s (−63%)** |
| Đóng gói tệp JAR với Gradle | 1s | 2s | ~0 |
| Lưu trữ thành phẩm JAR Artifact | 3s | 3s | 0 |
| Post Thiết lập môi trường Java JDK 17 | 5s | 1s | −4s (không cần lưu lại cache) |
| **Job Run Unit Tests** | **1m 5s** | **37s** | **−28s (−43%)** |
| Tổng thời gian workflow run | 1m 9s | 40s | −29s (−42%) |

![Hai lượt chạy trên tab Actions](images/01-actions-runs.png)

**Run #1 (chưa có cache):**

![Run #1 cold](images/02-run1-cold.png)

**Run #2 (đã có cache):**

![Run #2 warm](images/03-run2-warm.png)

### Log khôi phục cache ở Run #2 (step `Thiết lập môi trường Java JDK 17`)

```text
Cache hit for: setup-java-Linux-x64-gradle-543372610335759a235464acfe1d23459bde5f735e32ebdb28dd97589bcfda63
Received 80980856 of 80980856 (100.0%), 46.8 MBs/sec
Cache Size: ~77 MB (80980856 B)
Cache restored successfully
Cache restored from key: setup-java-Linux-x64-gradle-543372610335759a235464acfe1d23459bde5f735e32ebdb28dd97589bcfda63
Cache hit for: setup-java-Linux-x64-gradle-wrapper-cb10f63b6e80d04c47ba63d0097b1be1d92513baebbf33fb9b91e65214965f4c
Received 150685583 of 150685583 (100.0%), 48.7 MBs/sec
Cache Size: ~144 MB (150685583 B)
Cache restored successfully
gradle-wrapper cache restored from key: setup-java-Linux-x64-gradle-wrapper-cb10f63b6e80d04c47ba63d0097b1be1d92513baebbf33fb9b91e65214965f4c
```

Có **2 gói cache** được khôi phục thành công:

| Gói cache | Khoá | Dung lượng | Nội dung |
|---|---|---|---|
| Dependency | `setup-java-Linux-x64-gradle-5433…` | ~77 MB | `~/.gradle/caches`: các thư viện Spring Boot, Hibernate, JUnit, H2… |
| Gradle Wrapper | `setup-java-Linux-x64-gradle-wrapper-cb10…` | ~144 MB | `~/.gradle/wrapper`: bản phân phối Gradle mà `./gradlew` cần |

Tổng ~221 MB được tải từ hạ tầng cache của GitHub với tốc độ ~47 MB/s, thay cho việc tải lại từ Maven Central và services.gradle.org.

![Cache restored successfully](images/04-cache-restored.png)

## 4. Phân tích

- **Step setup-java chậm hơn 5 giây ở run #2** vì nó phải tải và giải nén gói cache. Đây là chi phí bỏ ra để tiết kiệm ở bước sau.
- **Step `./gradlew test` nhanh hơn 31 giây (49s xuống 18s).** Gradle Wrapper và dependency (Spring Boot, Hibernate, JUnit, H2...) đã có sẵn trong `~/.gradle`, không phải tải lại từ Maven Central. 18 giây còn lại là thời gian thật để khởi động Gradle daemon, biên dịch và chạy test.
- **Step `Post` setup-java nhanh hơn 4 giây** vì cache đã tồn tại với cùng khoá nên không cần nén và lưu lại.
- **Tổng cộng job giảm 43%.** Với dự án lớn hơn, nhiều dependency hơn hoặc mạng runner chậm, mức tiết kiệm còn rõ hơn.

## 5. Kết luận

- Chỉ cần thêm một dòng `cache: 'gradle'`, workflow tự lưu và khôi phục thư mục cache Gradle giữa các lượt chạy.
- Lượt chạy thứ hai rút thời gian job từ **1m 5s xuống 37s** và bước test từ **49s xuống 18s**.
- Cache chỉ bị làm mới khi `build.gradle` hoặc phiên bản Gradle thay đổi, nên các lần push thường ngày đều được hưởng lợi.
