# Playwright + Spring MVC + Microsoft Entra ID — Acceptance Test Scaffold

## Project Structure

```
src/test/
├── java/com/example/test/
│   ├── config/
│   │   └── TestConfig.java            # Environment-aware config loader
│   ├── auth/
│   │   └── EntraAuthHelper.java       # Reusable Entra ID login utility
│   ├── base/
│   │   ├── PlaywrightLifecycle.java    # JUnit 5 extension for browser lifecycle
│   │   ├── BaseAcceptanceTest.java     # Abstract base all tests extend
│   │   └── ScreenshotOnFailure.java   # Auto-screenshot on test failure
│   └── acceptance/
│       ├── DashboardTest.java          # Sample test
│       └── AdminPageTest.java          # Sample multi-role test
└── resources/
    ├── env/
    │   ├── dev.properties
    │   ├── staging.properties
    │   └── uat.properties
    └── logback-test.xml
```

---

## 1. pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>acceptance-tests</artifactId>
    <version>1.0.0-SNAPSHOT</version>

    <properties>
        <java.version>17</java.version>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
        <playwright.version>1.49.0</playwright.version>
        <junit.version>5.11.3</junit.version>

        <!-- Default environment — override with -Dtest.env=staging -->
        <test.env>dev</test.env>
    </properties>

    <dependencies>
        <dependency>
            <groupId>com.microsoft.playwright</groupId>
            <artifactId>playwright</artifactId>
            <version>${playwright.version}</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <version>${junit.version}</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-api</artifactId>
            <version>2.0.16</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>ch.qos.logback</groupId>
            <artifactId>logback-classic</artifactId>
            <version>1.5.12</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <testResources>
            <testResource>
                <directory>src/test/resources</directory>
                <filtering>true</filtering>
            </testResource>
        </testResources>

        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>3.5.2</version>
                <configuration>
                    <systemPropertyVariables>
                        <test.env>${test.env}</test.env>
                    </systemPropertyVariables>
                    <includes>
                        <include>**/*Test.java</include>
                        <include>**/*IT.java</include>
                    </includes>
                </configuration>
            </plugin>

            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-failsafe-plugin</artifactId>
                <version>3.5.2</version>
                <configuration>
                    <systemPropertyVariables>
                        <test.env>${test.env}</test.env>
                    </systemPropertyVariables>
                </configuration>
                <executions>
                    <execution>
                        <goals>
                            <goal>integration-test</goal>
                            <goal>verify</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>

    <!-- Optional Maven profiles as shorthand -->
    <profiles>
        <profile>
            <id>dev</id>
            <properties><test.env>dev</test.env></properties>
        </profile>
        <profile>
            <id>staging</id>
            <properties><test.env>staging</test.env></properties>
        </profile>
        <profile>
            <id>uat</id>
            <properties><test.env>uat</test.env></properties>
        </profile>
    </profiles>
</project>
```

---

## 2. Environment Property Files

### `src/test/resources/env/dev.properties`

```properties
# Application under test
app.base-url=https://myapp-dev.example.com

# Entra ID / Microsoft login
entra.login-url=https://login.microsoftonline.com
entra.tenant-id=YOUR_DEV_TENANT_ID

# Test user credentials (prefer env vars or vault for CI)
test.user.email=${TEST_USER_EMAIL:dev-testuser@example.com}
test.user.password=${TEST_USER_PASSWORD:}

# Playwright settings
playwright.headless=true
playwright.slow-mo=0
playwright.timeout=30000
```

### `src/test/resources/env/staging.properties`

```properties
app.base-url=https://myapp-staging.example.com

entra.login-url=https://login.microsoftonline.com
entra.tenant-id=YOUR_STAGING_TENANT_ID

test.user.email=${TEST_USER_EMAIL:staging-testuser@example.com}
test.user.password=${TEST_USER_PASSWORD:}

playwright.headless=true
playwright.slow-mo=0
playwright.timeout=30000
```

### `src/test/resources/env/uat.properties`

```properties
app.base-url=https://myapp-uat.example.com

entra.login-url=https://login.microsoftonline.com
entra.tenant-id=YOUR_UAT_TENANT_ID

test.user.email=${TEST_USER_EMAIL:uat-testuser@example.com}
test.user.password=${TEST_USER_PASSWORD:}

playwright.headless=true
playwright.slow-mo=100
playwright.timeout=45000
```

---

## 3. TestConfig.java — Environment-Aware Configuration Loader

```java
package com.example.test.config;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.io.IOException;
import java.io.InputStream;
import java.util.Properties;

/**
 * Loads environment-specific test configuration.
 *
 * Resolution order for each property value:
 *   1. JVM system property   (-Dtest.user.email=...)
 *   2. OS environment variable (TEST_USER_EMAIL)
 *   3. Value in env/{env}.properties file
 *
 * The target environment is chosen by system property "test.env"
 * (passed via Maven: -Dtest.env=staging).  Defaults to "dev".
 */
public final class TestConfig {

    private static final Logger log = LoggerFactory.getLogger(TestConfig.class);
    private static final TestConfig INSTANCE = new TestConfig();

    private final Properties props = new Properties();
    private final String environment;

    private TestConfig() {
        this.environment = resolveEnv();
        loadPropertiesFile();
        overrideFromSystemAndEnv();
        log.info("TestConfig loaded for environment: {}", environment);
    }

    public static TestConfig get() {
        return INSTANCE;
    }

    // ---- Accessors ----

    public String env()              { return environment; }
    public String baseUrl()          { return require("app.base-url"); }
    public String entraLoginUrl()    { return require("entra.login-url"); }
    public String entraTenantId()    { return require("entra.tenant-id"); }
    public String testUserEmail()    { return require("test.user.email"); }
    public String testUserPassword() { return require("test.user.password"); }

    public boolean headless() {
        return Boolean.parseBoolean(props.getProperty("playwright.headless", "true"));
    }

    public int slowMo() {
        return Integer.parseInt(props.getProperty("playwright.slow-mo", "0"));
    }

    public int timeout() {
        return Integer.parseInt(props.getProperty("playwright.timeout", "30000"));
    }

    public String property(String key) {
        return props.getProperty(key);
    }

    public String property(String key, String defaultValue) {
        return props.getProperty(key, defaultValue);
    }

    // ---- Internals ----

    private String resolveEnv() {
        String env = System.getProperty("test.env");
        if (env == null || env.isBlank()) {
            env = System.getenv("TEST_ENV");
        }
        if (env == null || env.isBlank()) {
            env = "dev";
        }
        return env.trim().toLowerCase();
    }

    private void loadPropertiesFile() {
        String path = "env/" + environment + ".properties";
        try (InputStream is = getClass().getClassLoader().getResourceAsStream(path)) {
            if (is == null) {
                throw new IllegalStateException(
                    "No config file found for environment '%s' at classpath:%s"
                        .formatted(environment, path));
            }
            props.load(is);
        } catch (IOException e) {
            throw new IllegalStateException("Failed to load " + path, e);
        }
    }

    /**
     * For every loaded property, check if a JVM system property or OS env
     * var provides an override.
     *
     * Env var names are derived by uppercasing and replacing dots/hyphens
     * with underscores:  test.user.email  →  TEST_USER_EMAIL
     */
    private void overrideFromSystemAndEnv() {
        for (String key : props.stringPropertyNames()) {
            // 1. System property wins
            String sysProp = System.getProperty(key);
            if (sysProp != null && !sysProp.isBlank()) {
                props.setProperty(key, sysProp);
                continue;
            }

            // 2. Env var
            String envKey = key.toUpperCase().replaceAll("[.\\-]", "_");
            String envVal = System.getenv(envKey);
            if (envVal != null && !envVal.isBlank()) {
                props.setProperty(key, envVal);
            }

            // 3. Resolve ${ENV_VAR:default} placeholders in property values
            String value = props.getProperty(key);
            if (value != null && value.startsWith("${") && value.contains(":")) {
                String inner = value.substring(2, value.length() - 1);
                String[] parts = inner.split(":", 2);
                String resolved = System.getenv(parts[0]);
                props.setProperty(key,
                    (resolved != null && !resolved.isBlank()) ? resolved : parts[1]);
            }
        }
    }

    private String require(String key) {
        String value = props.getProperty(key);
        if (value == null || value.isBlank()) {
            throw new IllegalStateException(
                "Required test config property '%s' is missing for env '%s'"
                    .formatted(key, environment));
        }
        return value;
    }
}
```

---

## 4. EntraAuthHelper.java — Reusable Entra ID Login Utility

```java
package com.example.test.auth;

import com.example.test.config.TestConfig;
import com.microsoft.playwright.*;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;

/**
 * Automates Microsoft Entra ID (Azure AD) login via the browser.
 *
 * Workflow:
 *   1. Navigate to the app's protected URL → Spring Security redirects to Entra.
 *   2. Fill email on Microsoft's sign-in page.
 *   3. Fill password and submit.
 *   4. Handle the "Stay signed in?" prompt if it appears.
 *   5. Wait for redirect back to the application.
 *   6. Save browser storage state so subsequent tests skip login.
 *
 * The saved state file is reused across tests in the same run,
 * so only the first test pays the login cost.
 */
public final class EntraAuthHelper {

    private static final Logger log = LoggerFactory.getLogger(EntraAuthHelper.class);

    /** Directory where auth state files are cached per environment. */
    private static final Path AUTH_STATE_DIR = Paths.get("target", "auth-state");

    private final TestConfig config;

    public EntraAuthHelper() {
        this(TestConfig.get());
    }

    public EntraAuthHelper(TestConfig config) {
        this.config = config;
    }

    // ----------------------------------------------------------------
    //  Public API
    // ----------------------------------------------------------------

    /**
     * Returns a path to a cached auth-state JSON file for the current
     * env + user.  If the file doesn't exist yet, performs a fresh login.
     */
    public Path getOrCreateAuthState(Browser browser) {
        Path stateFile = authStateFile(config.testUserEmail());
        if (Files.exists(stateFile)) {
            log.info("Reusing cached auth state: {}", stateFile);
            return stateFile;
        }
        return performLogin(browser, stateFile);
    }

    /**
     * Creates a new BrowserContext that is already authenticated.
     */
    public BrowserContext newAuthenticatedContext(Browser browser) {
        Path stateFile = getOrCreateAuthState(browser);
        return browser.newContext(
            new Browser.NewContextOptions().setStorageStatePath(stateFile));
    }

    // ----------------------------------------------------------------
    //  Login flow
    // ----------------------------------------------------------------

    /**
     * Performs the full Entra ID interactive login and saves browser state.
     */
    public Path performLogin(Browser browser, Path stateFile) {
        log.info("Performing Entra ID login for user: {}", config.testUserEmail());

        try {
            Files.createDirectories(stateFile.getParent());
        } catch (Exception e) {
            throw new RuntimeException("Cannot create auth state directory", e);
        }

        BrowserContext context = browser.newContext();
        Page page = context.newPage();
        page.setDefaultTimeout(config.timeout());

        try {
            // 1. Navigate to the app — Spring Security redirects to Entra
            page.navigate(config.baseUrl());
            log.debug("Navigated to {}, waiting for Entra redirect...",
                       config.baseUrl());

            // 2. Wait for the Microsoft login page
            page.waitForURL(
                url -> url.contains("login.microsoftonline.com")
                    || url.contains("login.live.com"),
                new Page.WaitForURLOptions().setTimeout(config.timeout()));

            // 3. Enter email
            fillEmail(page);

            // 4. Enter password
            fillPassword(page);

            // 5. Handle "Stay signed in?" / consent prompts
            handlePostLoginPrompts(page);

            // 6. Wait for redirect back to the app
            page.waitForURL(
                url -> url.startsWith(config.baseUrl()),
                new Page.WaitForURLOptions().setTimeout(config.timeout()));
            log.info("Login successful — redirected back to app.");

            // 7. Persist cookies + localStorage
            context.storageState(
                new BrowserContext.StorageStateOptions().setPath(stateFile));
            log.info("Auth state saved to {}", stateFile);

            return stateFile;

        } catch (Exception e) {
            // Capture a screenshot for debugging
            try {
                Path screenshot = AUTH_STATE_DIR.resolve("login-failure.png");
                page.screenshot(new Page.ScreenshotOptions()
                    .setPath(screenshot).setFullPage(true));
                log.error("Login failed — screenshot: {}", screenshot);
            } catch (Exception ignored) { }

            throw new RuntimeException(
                "Entra ID login failed for " + config.testUserEmail(), e);
        } finally {
            context.close();
        }
    }

    // ----------------------------------------------------------------
    //  Private step helpers
    // ----------------------------------------------------------------

    private void fillEmail(Page page) {
        Locator emailInput = page.locator(
            "input[type='email'], input[name='loginfmt']");
        emailInput.waitFor(
            new Locator.WaitForOptions().setTimeout(config.timeout()));
        emailInput.fill(config.testUserEmail());

        page.locator("input[type='submit'], button[type='submit']").click();
        log.debug("Email entered, clicked Next.");

        // Wait for the page to transition to the password step
        page.waitForTimeout(1500);
    }

    private void fillPassword(Page page) {
        Locator passwordInput = page.locator(
            "input[type='password'], input[name='passwd']");
        passwordInput.waitFor(
            new Locator.WaitForOptions().setTimeout(config.timeout()));
        passwordInput.fill(config.testUserPassword());

        page.locator("input[type='submit'], button[type='submit']").click();
        log.debug("Password entered, clicked Sign in.");

        page.waitForTimeout(2000);
    }

    private void handlePostLoginPrompts(Page page) {
        // "Stay signed in?" — click Yes
        Locator staySignedIn = page.locator(
            "#idSIButton9, input[value='Yes']");
        try {
            staySignedIn.waitFor(
                new Locator.WaitForOptions().setTimeout(5000));
            staySignedIn.click();
            log.debug("Handled 'Stay signed in?' prompt.");
        } catch (TimeoutError e) {
            log.debug("No 'Stay signed in?' prompt detected, continuing.");
        }

        // NOTE: if your tenant uses MFA with Authenticator push, you'll
        // need additional handling here.  For test tenants, disabling MFA
        // or using TOTP-based MFA (automatable) is recommended.
    }

    private Path authStateFile(String email) {
        String safe = email.replaceAll("[^a-zA-Z0-9]", "_");
        return AUTH_STATE_DIR.resolve(config.env() + "_" + safe + ".json");
    }
}
```

---

## 5. PlaywrightLifecycle.java — JUnit 5 Extension

```java
package com.example.test.base;

import com.example.test.config.TestConfig;
import com.microsoft.playwright.*;
import org.junit.jupiter.api.extension.*;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
 * JUnit 5 extension that manages the Playwright → Browser lifecycle.
 *
 * Playwright and Browser are expensive to create, so they are shared
 * across all tests (BeforeAll / AfterAll at the class level).
 */
public class PlaywrightLifecycle
        implements BeforeAllCallback, AfterAllCallback {

    private static final Logger log =
        LoggerFactory.getLogger(PlaywrightLifecycle.class);

    private static final ExtensionContext.Namespace NS =
        ExtensionContext.Namespace.create(PlaywrightLifecycle.class);

    @Override
    public void beforeAll(ExtensionContext context) {
        TestConfig config = TestConfig.get();

        Playwright playwright = Playwright.create();
        store(context).put("playwright", playwright);

        BrowserType.LaunchOptions opts = new BrowserType.LaunchOptions()
            .setHeadless(config.headless())
            .setSlowMo(config.slowMo());

        Browser browser = playwright.chromium().launch(opts);
        store(context).put("browser", browser);

        log.info("Playwright browser launched (headless={}, slowMo={})",
                 config.headless(), config.slowMo());
    }

    @Override
    public void afterAll(ExtensionContext context) {
        Browser browser = (Browser) store(context).get("browser");
        if (browser != null) browser.close();

        Playwright pw = (Playwright) store(context).get("playwright");
        if (pw != null) pw.close();

        log.info("Playwright browser closed.");
    }

    public static Browser getBrowser(ExtensionContext context) {
        return (Browser) store(context).get("browser");
    }

    private static ExtensionContext.Store store(ExtensionContext ctx) {
        return ctx.getRoot().getStore(NS);
    }
}
```

---

## 6. BaseAcceptanceTest.java — Abstract Base Class

```java
package com.example.test.base;

import com.example.test.auth.EntraAuthHelper;
import com.example.test.config.TestConfig;
import com.microsoft.playwright.*;
import org.junit.jupiter.api.*;
import org.junit.jupiter.api.extension.ExtendWith;
import org.junit.jupiter.api.extension.ExtensionContext;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.nio.file.Paths;

/**
 * Base class for all Playwright acceptance tests.
 *
 * Provides:
 *   - Environment-aware configuration via TestConfig
 *   - Automatic Entra ID login via EntraAuthHelper
 *   - A fresh, authenticated Page per test method
 *   - Automatic screenshot capture after every test
 *
 * Subclasses just write tests — no login boilerplate needed.
 */
@ExtendWith(PlaywrightLifecycle.class)
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
public abstract class BaseAcceptanceTest {

    private static final Logger log =
        LoggerFactory.getLogger(BaseAcceptanceTest.class);

    protected TestConfig config;
    protected Browser browser;
    protected EntraAuthHelper authHelper;

    // Per-test instances
    protected BrowserContext context;
    protected Page page;

    @BeforeAll
    void initShared(ExtensionContext extensionContext) {
        config    = TestConfig.get();
        browser   = PlaywrightLifecycle.getBrowser(extensionContext);
        authHelper = new EntraAuthHelper(config);

        // Eagerly perform login so the auth state file is ready
        authHelper.getOrCreateAuthState(browser);
    }

    @BeforeEach
    void setUp() {
        // Each test gets an isolated, pre-authenticated context
        context = authHelper.newAuthenticatedContext(browser);
        page    = context.newPage();
        page.setDefaultTimeout(config.timeout());
        log.debug("New authenticated page created for test.");
    }

    @AfterEach
    void tearDown(TestInfo testInfo) {
        if (page != null) {
            try {
                page.screenshot(new Page.ScreenshotOptions()
                    .setPath(Paths.get("target", "screenshots",
                        testInfo.getDisplayName() + ".png"))
                    .setFullPage(true));
            } catch (Exception ignored) { }
            page.close();
        }
        if (context != null) {
            context.close();
        }
    }

    // ---- Convenience helpers ----

    /** Navigate to a path relative to the app base URL. */
    protected void navigateTo(String path) {
        String url = config.baseUrl()
            + (path.startsWith("/") ? path : "/" + path);
        page.navigate(url);
    }

    /** Wait for the URL to contain a given substring. */
    protected void waitForPath(String pathContains) {
        page.waitForURL(url -> url.contains(pathContains));
    }
}
```

---

## 7. ScreenshotOnFailure.java — TestWatcher Extension

```java
package com.example.test.base;

import com.microsoft.playwright.Page;
import org.junit.jupiter.api.extension.*;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;

/**
 * Captures a full-page screenshot only when a test fails.
 * Looks for a field named "page" (type Page) on the test instance.
 *
 * Register: @ExtendWith(ScreenshotOnFailure.class)
 */
public class ScreenshotOnFailure implements TestWatcher {

    private static final Logger log =
        LoggerFactory.getLogger(ScreenshotOnFailure.class);

    private static final Path DIR = Paths.get("target", "screenshots");

    @Override
    public void testFailed(ExtensionContext ctx, Throwable cause) {
        ctx.getTestInstance().ifPresent(instance -> {
            try {
                Field f = findPageField(instance.getClass());
                if (f == null) return;

                f.setAccessible(true);
                Page page = (Page) f.get(instance);
                if (page == null || page.isClosed()) return;

                Files.createDirectories(DIR);
                String name = ctx.getDisplayName()
                    .replaceAll("[^a-zA-Z0-9_\\-]", "_");
                Path dest = DIR.resolve(name + "_FAILED.png");

                page.screenshot(new Page.ScreenshotOptions()
                    .setPath(dest).setFullPage(true));
                log.info("Failure screenshot saved: {}", dest);
            } catch (Exception e) {
                log.warn("Could not capture failure screenshot", e);
            }
        });
    }

    private Field findPageField(Class<?> clazz) {
        while (clazz != null) {
            try { return clazz.getDeclaredField("page"); }
            catch (NoSuchFieldException e) { clazz = clazz.getSuperclass(); }
        }
        return null;
    }
}
```

---

## 8. Sample Tests

### DashboardTest.java

```java
package com.example.test.acceptance;

import com.example.test.base.BaseAcceptanceTest;
import com.example.test.base.ScreenshotOnFailure;
import org.junit.jupiter.api.*;
import org.junit.jupiter.api.extension.ExtendWith;

import static com.microsoft.playwright.assertions.PlaywrightAssertions.assertThat;

@ExtendWith(ScreenshotOnFailure.class)
@DisplayName("Dashboard Acceptance Tests")
class DashboardTest extends BaseAcceptanceTest {

    @Test
    @DisplayName("Authenticated user sees the dashboard")
    void shouldDisplayDashboardAfterLogin() {
        navigateTo("/dashboard");

        assertThat(page).hasURL(config.baseUrl() + "/dashboard");
        assertThat(page.locator("h1")).containsText("Dashboard");
    }

    @Test
    @DisplayName("Dashboard shows the logged-in user's name")
    void shouldShowUserName() {
        navigateTo("/dashboard");

        assertThat(page.locator("[data-testid='user-name']")).isVisible();
    }

    @Test
    @DisplayName("Navigation menu is accessible")
    void shouldHaveNavigationMenu() {
        navigateTo("/dashboard");

        assertThat(page.locator("nav")).isVisible();
        assertThat(page.locator("nav a")).hasCount(5); // adjust to your app
    }
}
```

### AdminPageTest.java — Multi-Role Example

```java
package com.example.test.acceptance;

import com.example.test.base.BaseAcceptanceTest;
import com.example.test.base.ScreenshotOnFailure;
import org.junit.jupiter.api.*;
import org.junit.jupiter.api.extension.ExtendWith;

import static com.microsoft.playwright.assertions.PlaywrightAssertions.assertThat;

@ExtendWith(ScreenshotOnFailure.class)
@DisplayName("Admin Page Acceptance Tests")
class AdminPageTest extends BaseAcceptanceTest {

    @Test
    @DisplayName("Regular user cannot access admin page")
    void regularUserShouldNotAccessAdmin() {
        navigateTo("/admin");
        assertThat(page.locator("body")).containsText("Access Denied");
    }

    @Test
    @DisplayName("Search feature works")
    void searchFeatureShouldWork() {
        navigateTo("/search");

        page.locator("input[name='query']").fill("test query");
        page.locator("button[type='submit']").click();

        page.locator(".search-results").waitFor();
        assertThat(page.locator(".search-results .result-item"))
            .hasCount(10);
    }

    /**
     * To test with a different role, build a second EntraAuthHelper
     * with admin credentials loaded from config
     * (e.g. test.admin.email / test.admin.password).
     */
    @Test
    @DisplayName("Admin user can access admin page")
    @Disabled("Enable when admin test credentials are configured")
    void adminUserShouldAccessAdmin() {
        // BrowserContext adminCtx = new EntraAuthHelper(adminConfig)
        //         .newAuthenticatedContext(browser);
        // Page adminPage = adminCtx.newPage();
        // adminPage.navigate(config.baseUrl() + "/admin");
        // assertThat(adminPage.locator("h1")).containsText("Admin Panel");
        // adminCtx.close();
    }
}
```

---

## 9. logback-test.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <logger name="com.example.test" level="DEBUG"/>
    <logger name="com.microsoft.playwright" level="WARN"/>

    <root level="INFO">
        <appender-ref ref="STDOUT"/>
    </root>
</configuration>
```

---

## How to Run

```bash
# Default (dev environment)
mvn test

# Target a specific environment
mvn test -Dtest.env=staging
mvn test -Dtest.env=uat

# Using Maven profiles
mvn test -Pstaging

# Headed mode for debugging
mvn test -Dtest.env=dev -Dplaywright.headless=false

# Pass credentials via env vars (recommended for CI)
TEST_USER_EMAIL=me@corp.com TEST_USER_PASSWORD=secret mvn test -Dtest.env=staging

# Run a single test class
mvn test -Dtest.env=staging -Dtest=DashboardTest

# Integration tests via failsafe
mvn verify -Dtest.env=uat
```

---

## How It All Fits Together

1. **`-Dtest.env=staging`** is passed on the CLI → Maven Surefire forwards it as a system property.

2. **`TestConfig`** reads `env/staging.properties` from the classpath, then layers overrides from system properties and environment variables. CI pipelines inject secrets via env vars without touching config files.

3. **`PlaywrightLifecycle`** (JUnit 5 extension) launches Chromium once per test class using settings from `TestConfig`.

4. **`BaseAcceptanceTest.initShared()`** calls `EntraAuthHelper.getOrCreateAuthState()` which either reuses a cached session or performs a fresh login. The login flow navigates to your app, follows the Spring Security redirect to Microsoft's login page, fills email → password → handles "Stay signed in?", waits for the redirect back, and saves the browser's storage state (cookies + localStorage) to `target/auth-state/{env}_{user}.json`.

5. **Each `@Test` method** gets a fresh `BrowserContext` pre-loaded with that saved auth state — instant authenticated session, zero login overhead.

6. **`ScreenshotOnFailure`** captures a screenshot whenever a test fails, saved under `target/screenshots/`.

## Adapting to Your Setup

- **Selectors**: The Entra login selectors (`input[name='loginfmt']`, `input[name='passwd']`, `#idSIButton9`) match Microsoft's standard pages. Adjust if your tenant uses custom branding.

- **MFA**: For test tenants, disable MFA or use TOTP-based MFA and automate code generation in `handlePostLoginPrompts()`.

- **Multiple roles**: Add `test.admin.email` / `test.admin.password` to your properties files and create a second `EntraAuthHelper` instance with those credentials.

- **Session expiry**: If tests run long enough for sessions to expire, delete the cached state file and re-login by calling `performLogin()` directly.
