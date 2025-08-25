# Maven Multi-Service Monorepo Buildpack for Heroku

**A specialized preparation buildpack designed specifically for multi-service Maven projects with hierarchical structure (parent-child POM relationships).**

This buildpack is purpose-built for complex Maven monorepos that contain multiple services sharing common modules through a parent POM hierarchy. It solves the challenge of deploying individual services from Maven multi-module projects by intelligently preparing the monorepo structure so that Heroku's native Java buildpack can handle the compilation seamlessly.

## Target Use Case

**🎯 This buildpack is specifically designed for:**
- Maven multi-module projects with parent POM hierarchies
- Multiple microservices sharing common/shared modules
- Complex dependency management through `<dependencyManagement>` sections
- Spring Boot applications in monorepo structures
- Projects where services reference shared modules as Maven dependencies

**❌ This buildpack is NOT suitable for:**
- Simple single-service projects
- Node.js monorepos (use a Node.js-specific buildpack instead)
- Gradle-based projects
- Standalone Maven projects without shared dependencies

## How It Works

This is a **preparation-only buildpack** that runs *before* the standard Heroku Java buildpack:

1. **🔧 Structure Preparation**: Moves your target application (`APP_BASE`) to the build root
2. **📦 Shared Integration**: Copies shared module source code into your application's package structure  
3. **🔗 Smart Dependencies**: Automatically merges required dependencies from shared modules into your main POM
4. **🧹 Conflict Resolution**: Removes self-referencing shared module dependencies to prevent build errors
5. **⚙️ Build Setup**: Configures JAR naming and Java version for the subsequent Java buildpack
6. **🚀 Handoff**: Passes control to Heroku's Java buildpack for actual Maven compilation

**Result**: Your monorepo application builds as if it were a standalone project, with all shared dependencies properly resolved.

This buildpack is inspired by and based on the excellent work from [heroku-buildpack-multi-procfile](https://github.com/heroku/heroku-buildpack-multi-procfile) and [heroku-buildpack-monorepo](https://github.com/lstoll/heroku-buildpack-monorepo). It extends their concepts with intelligent shared module integration and Maven dependency management.

## Credits

This buildpack builds upon the foundational work of:
- **Original heroku-buildpack-multi-procfile**: Andrew Gwozdziewycz <apg@heroku.com> and Cyril David <cyx@heroku.com>
- **heroku-buildpack-monorepo**: Lincoln Stoll <lstoll@heroku.com>

We've extended their concept to include shared folder support for modern monorepo deployments.

# Quick Start

1. **Configure your Heroku app** with the buildpacks in the correct order:
   ```bash
   heroku buildpacks:clear -a <your-app>
   heroku buildpacks:add https://github.com/gdemarcosSFDC/heroku-monorepo -a <your-app>
   heroku buildpacks:add heroku/java -a <your-app>
   ```

2. **Set environment variables** to identify your app and shared modules:
   ```bash
   heroku config:set APP_BASE=path/to/your/app -a <your-app>
   heroku config:set SHARED_BASE=path/to/shared/module -a <your-app>
   ```

3. **Deploy normally**:
   ```bash
   git push heroku main
   ```

The buildpack will automatically prepare your monorepo structure and hand off to the Java buildpack for compilation.

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `APP_BASE` | Yes | Relative path to your application directory within the monorepo |
| `SHARED_BASE` | Optional | Relative path to shared module directory to integrate |
| `MAVEN_FLATTEN_POM` | Optional | Set to `true` to remove parent POM references for complex hierarchies |

## Shared Module Integration

The buildpack provides intelligent integration of shared modules into your main application. Unlike simple file copying, it performs **source code integration** that makes shared modules appear as part of your main application.

### What It Does

1. **📂 Source Integration**: Copies shared Java source files into your app's `src/main/java` structure using the correct package hierarchy
2. **📋 Resource Integration**: Merges shared resources into your app's `src/main/resources`
3. **🔗 Dependency Merging**: Automatically extracts and merges required dependencies from the shared module's POM into your main POM
4. **🚫 Conflict Resolution**: Removes the shared module dependency from your main POM (since source code is now integrated)
5. **⚡ Smart Detection**: Only merges dependencies that are actually used (scans import statements)

### Example

**Before** (Monorepo Structure):
```
monorepo/
├── my-app/
│   ├── pom.xml (depends on shared-persistence:jar)
│   └── src/main/java/com/mycompany/app/
└── shared-persistence/
    ├── pom.xml (contains spring-kafka dependency)
    └── src/main/java/com/mycompany/shared/
```

**After** (Prepared for Heroku Java buildpack):
```
build-root/
├── pom.xml (spring-kafka dependency automatically added)
├── src/main/java/
│   ├── com/mycompany/app/ (original app code)
│   └── com/mycompany/shared/ (shared code integrated)
└── system.properties (Java version configured)
```

**Configuration**:
```bash
heroku config:set APP_BASE=my-app -a <your-app>
heroku config:set SHARED_BASE=shared-persistence -a <your-app>
```

## Maven Multi-Module Support

The buildpack automatically handles the complexities of Maven multi-module projects when deploying individual modules from a monorepo. Here's what happens behind the scenes:

### Automatic Transformations

- **🔍 Parent POM Resolution**: Automatically locates and copies parent POMs to resolve `<parent>` references
- **🧹 Module Cleanup**: Removes `<modules>` sections from parent POMs (since sibling modules aren't included in the build)
- **🔗 Dependency Management**: Preserves `<dependencyManagement>` sections for version control
- **☕ Java Version Modernization**: Automatically upgrades Java 8/1.8 settings to Java 21 for modern language features
- **🎯 JAR Configuration**: Sets up proper JAR naming using `<finalName>` for clean output names
- **📋 System Properties**: Creates `system.properties` to explicitly set Java runtime version

### Smart Dependency Merging

When using `SHARED_BASE`, the buildpack performs intelligent dependency resolution:

1. **🔍 Import Analysis**: Scans your application's Java files for imports from shared modules
2. **📦 Selective Merging**: Only adds dependencies from shared POMs that are actually used
3. **🚫 Conflict Prevention**: Removes shared module JAR dependencies (since source is integrated)
4. **⚡ Version Resolution**: Uses Spring Boot BOM for version management when possible
5. **🛡️ Duplicate Prevention**: Avoids adding dependencies that already exist in your main POM

### Build Process Integration

The buildpack prepares everything for Heroku's Java buildpack:
- ✅ **Standalone POM**: Your POM becomes self-contained with all required dependencies
- ✅ **Integrated Sources**: Shared module code appears in your application's package structure
- ✅ **Clean JAR Output**: Produces a JAR with a predictable name (e.g., `my-app.jar`)
- ✅ **Runtime Ready**: Final JAR is copied to the root directory for Procfile access

## Complete Example

### Monorepo Structure
```
my-monorepo/
├── pom.xml                          (parent with dependencyManagement)
├── gateway-service/
│   ├── pom.xml                      (depends on shared-persistence)
│   ├── src/main/java/com/company/gateway/
│   └── Procfile
├── processor-service/  
│   ├── pom.xml                      (depends on shared-persistence)
│   └── src/main/java/com/company/processor/
└── shared-persistence/
    ├── pom.xml                      (contains Spring Kafka, JPA dependencies)
    └── src/main/java/com/company/shared/
```

### Heroku App Configuration
```bash
# Setup buildpacks (order is critical!)
heroku buildpacks:clear -a my-gateway-app
heroku buildpacks:add https://github.com/gdemarcosSFDC/heroku-monorepo -a my-gateway-app
heroku buildpacks:add heroku/java -a my-gateway-app

# Configure paths
heroku config:set APP_BASE=gateway-service -a my-gateway-app
heroku config:set SHARED_BASE=shared-persistence -a my-gateway-app
```

### What Happens During Deployment
1. 📂 **Structure Preparation**: `gateway-service` → build root
2. 📦 **Source Integration**: `shared-persistence/src` → `src/main/java/com/company/shared/`
3. 🔍 **Dependency Analysis**: Scans for `import com.company.shared.*` statements
4. 🔗 **Smart Merging**: Adds Spring Kafka + JPA deps to gateway POM
5. 🚫 **Cleanup**: Removes `shared-persistence` JAR dependency
6. ⚙️ **Build Config**: Sets Java 21, configures JAR name
7. 🏗️ **Java Buildpack**: Takes over and builds the prepared project
8. 🚀 **Runtime**: JAR available at root for Procfile

## Advanced Configuration

### POM Flattening for Complex Hierarchies

If you encounter parent POM resolution issues, enable flattening:

```bash
heroku config:set MAVEN_FLATTEN_POM=true -a <your-app>
```

This removes all `<parent>` references and creates standalone POMs while preserving dependency management.

### Troubleshooting

#### Build Order Issues
Ensure buildpacks are in the correct order. This buildpack must run **before** the Java buildpack:

```bash
heroku buildpacks:clear -a <your-app>
heroku buildpacks:add https://github.com/gdemarcosSFDC/heroku-monorepo -a <your-app>
heroku buildpacks:add heroku/java -a <your-app>
```

#### Java Version Issues
The buildpack automatically upgrades Java 8/1.8 to Java 21. To override:

```bash
heroku config:set MAVEN_OPTS="-Dmaven.compiler.source=17 -Dmaven.compiler.target=17" -a <your-app>
```

#### Missing Dependencies
If shared module dependencies aren't being merged:
1. Check that your app code imports classes from the shared module
2. Verify the shared module's POM contains the required dependencies
3. Enable debug mode and check logs:
   ```bash
   heroku config:set BUILDPACK_DEBUG=true -a <your-app>
   ```

#### JAR Not Found at Runtime
The buildpack automatically copies JARs to the root directory. Your Procfile should reference:
```
web: java -Dserver.port=$PORT -jar your-app-name.jar
```

### Supported Project Types

- ✅ Spring Boot applications
- ✅ Maven multi-module projects  
- ✅ Maven projects with parent POMs
- ✅ Projects using shared libraries/modules
- ⚠️ Gradle projects (not currently supported)

## Quick Reference

### Minimal Setup
```bash
heroku buildpacks:add https://github.com/gdemarcosSFDC/heroku-monorepo -a <app>
heroku buildpacks:add heroku/java -a <app>
heroku config:set APP_BASE=my-service -a <app>
```

### With Shared Module
```bash
heroku config:set APP_BASE=my-service -a <app>
heroku config:set SHARED_BASE=shared-libs -a <app>
```

### For Complex Hierarchies
```bash
heroku config:set MAVEN_FLATTEN_POM=true -a <app>
```

---

## License

This work is based on the original heroku-buildpack-multi-procfile and heroku-buildpack-monorepo projects. Please refer to their respective repositories for licensing information.