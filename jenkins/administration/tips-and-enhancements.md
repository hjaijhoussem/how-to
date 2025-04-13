# Tips and Enhacements for Pipeline Optimazation

1. [Stash and Unstash](#stash-and-unstash)
2. [Pipeline Caching](#pipeline-caching)

## Stash and Unstash

Stash and Unstash are Jenkins pipeline features that allow you to save and restore files between different stages of your pipeline. This is particularly useful when you need to share build artifacts, dependencies, or other files across different stages or nodes.

### Basic Syntax

```groovy
// Stash files
stash name: 'my-stash', includes: 'path/to/files/**'

// Unstash files
unstash 'my-stash'
```

### Common Use Cases

1. **Sharing Build Artifacts**
```groovy
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
                stash name: 'build-artifacts', includes: 'target/*.jar'
            }
        }
        stage('Test') {
            steps {
                unstash 'build-artifacts'
                sh 'mvn test'
            }
        }
    }
}
```

2. **Sharing Dependencies**
```groovy
pipeline {
    agent any
    stages {
        stage('Install Dependencies') {
            steps {
                sh 'npm install'
                stash name: 'node_modules', includes: 'node_modules/**'
            }
        }
        stage('Build and Test') {
            agent {
                label 'test-node'
            }
            steps {
                unstash 'node_modules'
                sh 'npm run build'
                sh 'npm test'
            }
        }
    }
}
```

### Best Practices

1. **Be Specific with Includes**
   - Only stash what you need
   - Use specific patterns to avoid stashing unnecessary files
   ```groovy
   // Good
   stash name: 'configs', includes: 'src/main/resources/*.yml'
   
   // Avoid
   stash name: 'everything', includes: '**'
   ```

2. **Use Meaningful Names**
   - Name your stashes descriptively
   - Include the purpose in the name
   ```groovy
   // Good
   stash name: 'frontend-build', includes: 'dist/**'
   
   // Avoid
   stash name: 'stuff', includes: '**'
   ```

3. **Clean Up After Use**
   - Remove stashed files when no longer needed
   ```groovy
   stage('Cleanup') {
       steps {
           cleanWs()
       }
   }
   ```

### Limitations

- Stash/unstash operations can be slow for large files
- Maximum stash size is limited by Jenkins configuration
- Stashes are temporary and cleared when the pipeline completes

### Performance Tips

1. **Minimize Stash Size**
   - Exclude unnecessary files using `excludes` parameter
   ```groovy
   stash name: 'build', 
           includes: 'dist/**',
           excludes: 'dist/**/*.map,dist/**/*.log'
   ```

2. **Use Parallel Stashing**
   - Stash multiple directories in parallel when possible
   ```groovy
   parallel {
       stage('Stash Frontend') {
           steps {
               stash name: 'frontend', includes: 'frontend/dist/**'
           }
       }
       stage('Stash Backend') {
           steps {
               stash name: 'backend', includes: 'backend/target/**'
           }
       }
   }
   ```

### Troubleshooting

Common issues and solutions:

1. **Stash Not Found**
   - Verify the stash name matches exactly
   - Ensure the stash was created in a previous stage
   - Check if the pipeline was restarted (stashes are cleared)

2. **Performance Issues**
   - Reduce the size of stashed files
   - Use more specific include/exclude patterns
   - Consider using workspace copying for small files

3. **Permission Issues**
   - Ensure Jenkins has read/write permissions
   - Check file ownership in the workspace

## Pipeline Caching   

<img src="../images/Job casher plugin.png" alt="Job casher plugin"/>

The Job Cacher plugin provides an efficient way to cache build artifacts and dependencies between pipeline runs, significantly improving build times and reducing resource usage.

### Basic Usage

```groovy
stages {
    stage('Installing Dependencies') {
        options { timestamps() }
        steps {
            cache(maxCacheSize: 550, caches: [
                arbitraryFileCache(
                    cacheName: 'npm-dependency-cache',
                    cacheValidityDecidingFile: 'package-lock.json',
                    includes: '**/*',
                    path: 'node_modules'
                )
            ]) {
                sh 'node -v'
                sh 'npm install --no-audit'
            }
        }
    }
}
```

### Key Features

1. **Automatic Caching**
   - Caches specified directories automatically
   - Restores cache on subsequent builds
   - Handles cache invalidation automatically

2. **Cache Configuration Options**
```groovy
options {
    cache(
        include: ['path/to/cache/**'],    // Files to cache
        exclude: ['**/*.log'],            // Files to exclude
        key: 'my-custom-key',             // Custom cache key
        restoreKeys: ['fallback-key'],    // Fallback cache keys
        path: '/custom/cache/path'         // Custom cache location
    )
}
```

### Common Use Cases

1. **Node.js Projects**
```groovy
pipeline {
    agent any
    options {
        cache(include: ['node_modules/**'])
    }
    stages {
        stage('Install') {
            steps {
                sh 'npm ci'  // Uses package-lock.json for consistent installs
            }
        }
    }
}
```

2. **Maven Projects**
```groovy
pipeline {
    agent any
    options {
        cache(include: [
            '.m2/repository/**',  // Maven dependencies
            'target/**'           // Build artifacts
        ])
    }
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
    }
}
```

### Best Practices

1. **Cache Key Management**
   - Use meaningful cache keys
   - Include version information in keys
   ```groovy
   options {
       cache(
           key: "node-${NODE_VERSION}-${package.json.hash}"
       )
   }
   ```

2. **Selective Caching**
   - Cache only necessary files
   - Exclude large binary files
   - Use specific patterns
   ```groovy
   options {
       cache(
           include: ['src/**', 'config/**'],
           exclude: ['**/*.zip', '**/*.tar.gz']
       )
   }
   ```

3. **Cache Maintenance**
   - Set appropriate TTL (Time To Live)
   - Implement cache cleanup strategies
   - Monitor cache size

### Performance Optimization

1. **Parallel Caching**
```groovy
pipeline {
    agent any
    options {
        cache(include: ['frontend/node_modules/**'])
    }
    stages {
        stage('Build') {
            parallel {
                stage('Frontend') {
                    steps {
                        dir('frontend') {
                            sh 'npm install'
                        }
                    }
                }
                stage('Backend') {
                    options {
                        cache(include: ['backend/target/**'])
                    }
                    steps {
                        dir('backend') {
                            sh 'mvn clean package'
                        }
                    }
                }
            }
        }
    }
}
```

2. **Incremental Caching**
```groovy
options {
    cache(
        include: ['dist/**'],
        key: "build-${BUILD_NUMBER}",
        restoreKeys: ["build-${BUILD_NUMBER-1}"]
    )
}
```

### Troubleshooting

1. **Cache Miss Issues**
   - Verify cache key consistency
   - Check file permissions
   - Ensure cache directory exists
   - Validate include/exclude patterns

2. **Performance Issues**
   - Monitor cache size
   - Review cache hit rates
   - Optimize include/exclude patterns
   - Consider using incremental caching

3. **Common Errors**
   - "Cache not found": Check cache key and restore keys
   - "Permission denied": Verify Jenkins user permissions
   - "Cache corrupted": Clear cache and rebuild

### Configuration in Jenkins

1. **Global Configuration**
   - Set default cache location
   - Configure cache retention policies
   - Set maximum cache size
   - Define cleanup schedules

2. **Security Considerations**
   - Implement cache access controls
   - Secure cache storage
   - Monitor cache usage
   - Regular security audits
