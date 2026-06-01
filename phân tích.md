## Phần 2 - Thực thi

### 1. Kiến trúc giải pháp đề xuất

Hệ thống sử dụng `CustomUserDetailsService` để lấy thông tin người dùng từ database thay vì dùng tài khoản hard-code trong bộ nhớ. Khi người dùng gửi yêu cầu đăng nhập, Spring Security sẽ nhận username/password, sau đó gọi `CustomUserDetailsService` để tìm user tương ứng trong database.

Mật khẩu trong database không được lưu dưới dạng plain text mà được lưu dưới dạng hash bằng BCrypt. Khi đăng nhập, `PasswordEncoder` sẽ so sánh mật khẩu người dùng nhập với mật khẩu đã mã hóa trong database. Nếu khớp, Spring Security tạo đối tượng xác thực và cho phép người dùng truy cập hệ thống.

---

### 2. Các thành phần chính

| Thành phần | Vai trò |
|---|---|
| Client | Gửi yêu cầu đăng nhập gồm username và password |
| Spring Security Filter Chain | Chặn request đăng nhập và chuyển sang quá trình xác thực |
| AuthenticationManager | Điều phối quá trình xác thực |
| CustomUserDetailsService | Tìm user trong database theo username |
| UserRepository | Truy vấn bảng users |
| Database | Lưu username, password hash và roles |
| PasswordEncoder | So sánh raw password với password hash |
| SecurityContext | Lưu thông tin xác thực sau khi đăng nhập thành công |

---

### 3. Sơ đồ luồng xác thực

```mermaid
graph TD
    A[Client gửi username/password] --> B[Spring Security Filter Chain]
    B --> C[AuthenticationManager]
    C --> D[CustomUserDetailsService]
    D --> E[UserRepository]
    E --> F[(Database users)]
    F --> E
    E --> D
    D --> G[Trả về UserDetails gồm username, password hash, roles]
    G --> C
    C --> H[PasswordEncoder kiểm tra raw password với password hash]
    H --> I{Mật khẩu hợp lệ?}
    I -- Không --> J[Trả về lỗi đăng nhập]
    I -- Có --> K[Tạo Authentication Object]
    K --> L[Lưu Authentication vào SecurityContext]
    L --> M[Cho phép truy cập tài nguyên]
4. Giải thích luồng hoạt động

Bước 1: Người dùng gửi yêu cầu đăng nhập từ client với username và password.

Bước 2: Request đi qua Spring Security Filter Chain. Đây là nơi Spring Security bắt đầu xử lý bảo mật trước khi request được đi vào controller.

Bước 3: AuthenticationManager nhận thông tin đăng nhập và bắt đầu điều phối quá trình xác thực.

Bước 4: AuthenticationManager gọi CustomUserDetailsService.

Bước 5: CustomUserDetailsService gọi UserRepository để tìm user trong database theo username.

Bước 6: Nếu tìm thấy user, database trả về thông tin gồm username, password đã được hash bằng BCrypt và danh sách roles.

Bước 7: CustomUserDetailsService chuyển dữ liệu user thành đối tượng UserDetails và trả về cho Spring Security.

Bước 8: Spring Security dùng PasswordEncoder, cụ thể là BCryptPasswordEncoder, để kiểm tra mật khẩu người dùng nhập có khớp với password hash trong database hay không.

Bước 9: Nếu mật khẩu không hợp lệ, hệ thống trả về lỗi đăng nhập.

Bước 10: Nếu mật khẩu hợp lệ, Spring Security tạo Authentication Object.

Bước 11: Đối tượng xác thực được lưu vào SecurityContext.

Bước 12: Người dùng được xác thực thành công và có thể truy cập các tài nguyên phù hợp với role của mình.
5. Phác thảo code minh họa
CustomUserDetailsService
@Service
public class CustomUserDetailsService implements UserDetailsService {

    private final UserRepository userRepository;

    public CustomUserDetailsService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Override
    public UserDetails loadUserByUsername(String username) {
        AppUser user = userRepository.findByUsername(username)
                .orElseThrow(() -> new UsernameNotFoundException("User not found"));

        return User.builder()
                .username(user.getUsername())
                .password(user.getPassword())
                .roles(user.getRoles().split(","))
                .build();
    }
}
PasswordEncoder
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}
SecurityConfig
@Configuration
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
                .authorizeHttpRequests(auth -> auth
                        .requestMatchers("/login", "/register").permitAll()
                        .anyRequest().authenticated()
                )
                .formLogin(Customizer.withDefaults());

        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
6. Kết luận

Kiến trúc này giúp hệ thống quản lý người dùng linh hoạt hơn vì thông tin tài khoản được lấy trực tiếp từ database. Đồng thời, việc dùng BCryptPasswordEncoder giúp bảo vệ mật khẩu an toàn hơn, tránh việc lưu mật khẩu thật dưới dạng plain text. Spring Security kết hợp với CustomUserDetailsService và PasswordEncoder tạo thành một luồng xác thực đầy đủ, phù hợp cho hệ thống thư viện số có nhiều người dùng và nhiều vai trò khác nhau.