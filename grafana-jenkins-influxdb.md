After using the docker-compose file I provided, here are the exact settings you should use in Grafana's data source configuration:
```
# InfluxDB Data Source Settings in Grafana
URL: http://influxdb:8086
Database: junitdata
Query Language: InfluxQL
Basic auth: Enabled
With Credentials: Enabled
User: grafana
Password: grafana
HTTP Method: GET
```


Alternative settings (using admin credentials if the above doesn't work):
```
URL: http://influxdb:8086
Database: junitdata
Query Language: InfluxQL
Basic auth: Enabled
With Credentials: Enabled
User: admin
Password: admin
HTTP Method: GET
```

These values match the environment variables we set in the docker-compose.yml:

```yaml
environment:
  - INFLUXDB_DB=junitdata           # Database name
  - INFLUXDB_HTTP_AUTH_ENABLED=true
  - INFLUXDB_ADMIN_USER=admin       # Admin credentials
  - INFLUXDB_ADMIN_PASSWORD=admin
  - INFLUXDB_USER=grafana           # Regular user credentials
  - INFLUXDB_USER_PASSWORD=grafana
```

Try both the regular user (grafana/grafana) and admin (admin/admin) credentials to see which one works. The admin credentials might be needed if there are any permission issues.

Here's a step-by-step guide to integrate InfluxDB with Jenkins to store JUnit test results:

 Go to Jenkins → Manage Jenkins → Plugins → Available plugins
   Search and install:
   - "InfluxDB Plugin"
   - "JUnit Plugin" (probably already installed)

Configure InfluxDB in Jenkins:
   Go to Jenkins → Manage Jenkins → System
   Scroll to "InfluxDB targets"
   Click "Add InfluxDB target"
   Configure:
```
   - Description: Jenkins InfluxDB
   - URL: http://influxdb:8086
   - Database: junitdata
   - Username: grafana
   - Password: grafana
   - Retention Policy: autogen
```

Update your Jenkinsfile to include InfluxDB reporting:
```groovy
pipeline {
    // ... existing configuration ...
    
    environment {
        // ... existing environment variables ...
        
        // InfluxDB Configuration
        INFLUXDB_URL = 'http://influxdb:8086'
        INFLUXDB_DATABASE = 'junitdata'
        INFLUXDB_MEASUREMENT = 'junit_results'
    }

    stages {
        // ... your existing stages ...

        stage('Run Tests') {
            steps {
                sh 'npm test'  // or your test command
            }
            post {
                always {
                    junit(
                        allowEmptyResults: true,
                        testResults: 'test-results.xml',
                        skipPublishingChecks: true
                    )
                    step([
                        $class: 'InfluxDbPublisher',
                        selectedTarget: 'Jenkins InfluxDB',
                        customProjectName: env.JOB_NAME,
                        customBuildNumber: env.BUILD_NUMBER,
                        customDataMap: [
                            testSuite: env.JOB_NAME,
                            buildNumber: env.BUILD_NUMBER,
                            buildResult: currentBuild.result ?: 'SUCCESS'
                        ]
                    ])
                }
            }
        }
    }
    
    post {
        always {
            // Existing post actions...
            
            // Add InfluxDB reporting
            step([
                $class: 'InfluxDbPublisher',
                selectedTarget: 'Jenkins InfluxDB',
                customProjectName: env.JOB_NAME,
                customBuildNumber: env.BUILD_NUMBER,
                customDataMap: [
                    project: env.JOB_NAME,
                    buildNumber: env.BUILD_NUMBER,
                    buildResult: currentBuild.result ?: 'SUCCESS',
                    buildDuration: currentBuild.duration
                ]
            ])
        }
    }
}
```

Create a Grafana Dashboard:
   In Grafana:
   1. Click '+ Create' → 'Dashboard'
   2. Add a new panel
   3. Data source: Select your InfluxDB
   4. Query:    
```sql
FROM junit_results
SELECT count("tests")
GROUP BY build_number
```

Example InfluxDB Queries for Grafana:
```sql	
# Total Tests per Build
SELECT count("tests") FROM "junit_results" WHERE $timeFilter GROUP BY "build_number"

# Failed Tests per Build
SELECT count("failures") FROM "junit_results" WHERE $timeFilter GROUP BY "build_number"

# Test Duration Trend
SELECT mean("duration") FROM "junit_results" WHERE $timeFilter GROUP BY "build_number"
```
    
Full Example Dashboard Query:
```sql
SELECT 
  count("tests") as "Total Tests",
  count("failures") as "Failed Tests",
  count("skipped") as "Skipped Tests",
  mean("duration") as "Average Duration"
FROM "junit_results"
WHERE $timeFilter
GROUP BY time($__interval), "build_number"
```
```yaml
version: '3'

networks:
  monitoring:

services:
  jenkins:
    image: jenkins/jenkins:lts
    networks:
      - monitoring
    ports:
      - "8080:8080"
    volumes:
      - jenkins_home:/var/jenkins_home

  influxdb:
    image: influxdb:1.8
    networks:
      - monitoring
    ports:
      - "8086:8086"
    environment:
      - INFLUXDB_DB=junitdata
      - INFLUXDB_HTTP_AUTH_ENABLED=true
      - INFLUXDB_ADMIN_USER=admin
      - INFLUXDB_ADMIN_PASSWORD=admin
      - INFLUXDB_USER=grafana
      - INFLUXDB_USER_PASSWORD=grafana
    volumes:
      - influxdb-data:/var/lib/influxdb

  grafana:
    image: grafana/grafana
    networks:
      - monitoring
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
    depends_on:
      - influxdb

volumes:
  jenkins_home:
  influxdb-data:
  grafana-data:

networks:
  monitoring:
```
```shell
Verify Setup:
   docker exec -it influxdb influx -username grafana -password grafana
    > USE junitdata
    > SHOW MEASUREMENTS
    > SELECT * FROM junit_results ORDER BY time DESC LIMIT 5
``` 