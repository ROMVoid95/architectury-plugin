# Architect
Architect is a gradle plugin to allow easier multi-modloader set-ups using a common module.

This will only function with mojmap, also known as mojang mappings, as the MCP remapping is half-baked and will only remap classes. `Environment` and `EnvType` will be remapped to `OnlyIn` and `Dist`.

### How does it work
Fabric Side:

- Module `fabric` depends on `common`, which is shaded afterwards

Forge Side:

- Module `forge` depends on MCP remapped version of `common`, which is shaded afterwards
- A fake mod is generated, to let forge load it on the correct mod loader and let forge load its assets

### Usage
Your gradle version **MUST** be 5.5.1, all `assets` or `data` should go into the common module, with modloader specific files to the corresponding module.

settings.gradle
```groovy
pluginManagement {
    repositories {
        jcenter()
        maven { url "https://maven.fabricmc.net/" }
        maven { url "https://dl.bintray.com/shedaniel/cloth" }
        gradlePluginPortal()
    }
}

include("common")
include("fabric")
include("forge")
```

gradle.properties
```properties
minecraft_version=1.16.2

archives_base_name=modid
mod_version=1.0.0
maven_group=net.examplemod

fabric_loader_version=0.10.0+build.208
fabric_api_version=0.22.1+build.409-1.16

forge_version=33.0.61
```

build.gradle
```groovy
plugins {
    id "architect-plugin" version "1.0.6"
}

architect {
    minecraft = rootProject.minecraft_version
}

allprojects {
    apply plugin: "java"
    apply plugin: "architect-plugin"
    
    archivesBaseName = rootProject.archives_base_name
    version = rootProject.mod_version
    group = rootProject.maven_group

    tasks.withType(JavaCompile) {
        options.encoding = "UTF-8"
    }
}
```

common/build.gradle
```groovy
plugins {
    id "fabric-loom"
}

dependencies {
    minecraft "com.mojang:minecraft:${rootProject.architect.minecraft}"
    mappings minecraft.officialMojangMappings()
    // We depend on fabric loader here to use the fabric @Environment annotations
    // Do NOT use other classes from fabric loader
    modCompile "net.fabricmc:fabric-loader:${rootProject.fabric_loader_version}"
}

architect {
    common()
}
```

fabric/build.gradle
```groovy
plugins {
    id "fabric-loom"
    id "com.github.johnrengelman.shadow" version "5.0.0"
}

configurations {
    shadow
}

dependencies {
    minecraft("com.mojang:minecraft:${rootProject.architect.minecraft}")
    mappings(minecraft.officialMojangMappings())
    modCompile("net.fabricmc:fabric-loader:${rootProject.fabric_loader_version}")
    modCompile("net.fabricmc.fabric-api:fabric-api:${rootProject.fabric_api_version}")

    compile(project(":common")) {
        transitive = false
    }
    shadow(project(":common")) {
        transitive = false
    }
}

shadowJar {
    configurations = [project.configurations.shadow]
    classifier "shadow"
}

remapJar {
    dependsOn(shadowJar)
    input.set(shadowJar.archivePath)
}
```

forge/build.gradle
```groovy
buildscript {
    repositories {
        maven { url "https://files.minecraftforge.net/maven" }
        jcenter()
        mavenCentral()
    }
    dependencies {
        classpath(group: "net.minecraftforge.gradle", name: "ForgeGradle", version: "3.+", changing: true)
    }
}

plugins {
    id "com.github.johnrengelman.shadow" version "5.0.0"
    id "eclipse"
}

apply plugin: "net.minecraftforge.gradle"

minecraft {
    mappings(channel: "official", version: rootProject.architect.minecraft)
    accessTransformer = file('src/main/resources/META-INF/accesstransformer.cfg')
    runs {
        client {
            workingDirectory project.file("run")
            mods {
                examplemod {
                    source sourceSets.main
                }
            }
        }
        server {
            workingDirectory project.file("run")
            mods {
                examplemod {
                    source sourceSets.main
                }
            }
        }
    }
}

repositories {
    jcenter()
    maven { url "https://files.minecraftforge.net/maven" }
}

configurations {
    shadow
}

dependencies {
    minecraft("net.minecraftforge:forge:${rootProject.architect.minecraft}-${rootProject.forge_version}")

    compile(project(path: ":common", configuration: "mcpGenerateMod")) {
        transitive = false
    }
    shadow(project(path: ":common", configuration: "mcp")) {
        transitive = false
    }
}

shadowJar {
    exclude "fabric.mod.json"

    configurations = [project.configurations.shadow]
    classifier null
}

reobf {
    shadowJar {}
}
```

common/src/main/resources/fabric.mod.json
```json
{
  "_comment": "This file is here to make fabric loader load this on the Knot classloader.",
  "schemaVersion": 1,
  "id": "modid-common",
  "version": "0.0.1"
}
```