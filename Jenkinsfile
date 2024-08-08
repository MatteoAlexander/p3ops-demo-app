pipeline {
    agent any

    stages {
        stage('Delete REPO') {
            steps {
                sh 'if [ -d p3ops-demo-app ]; then rm -rf p3ops-demo-app; fi'
            }
        }

        stage('Git Checkout') {
            steps {
                sh 'git clone --branch main https://github.com/MatteoAlexander/p3ops-demo-app'
            }
        }

        stage('Check Docker Containers') {
            steps {
                script {
                    def dotnetContainerExists = sh(script: 'docker ps -q -f name=dotnet6-container', returnStatus: true) == 0
                    def sqlContainerExists = sh(script: 'docker ps -q -f name=sql-container', returnStatus: true) == 0
                    def jenkinsContainerExists = sh(script: 'docker ps -q -f name=jenkins', returnStatus: true) == 0

                    if (!dotnetContainerExists) {
                        error "dotnet6-container is not running"
                    }
                    if (!sqlContainerExists) {
                        error "sql-container is not running"
                    }
                    if (!jenkinsContainerExists) {
                        error "jenkins container is not running"
                    }
                }
            }
        }

        stage('Copy files to dotnet6-container') {
            steps {
                script {
                    sh 'docker cp /var/jenkins_home/workspace/dotnet/p3ops-demo-app dotnet6-container:/p3ops-demo-app'
                }
            }
        }

        stage('Verify copied files') {
            steps {
                script {
                    sh 'docker exec dotnet6-container ls -l /p3ops-demo-app'
                }
            }
        }

        stage('Install Node.js and npm') {
            steps {
                script {
                    sh 'docker exec dotnet6-container apt-get update'
                    sh 'docker exec dotnet6-container apt-get install -y nodejs npm'

                    // Assuming you need to install and build in the Client directory
                    sh 'docker exec dotnet6-container bash -c "cd /p3ops-demo-app/src/Client && npm install tailwindcss@2.2.19 --legacy-peer-deps"'
                    sh 'docker exec dotnet6-container bash -c "cd /p3ops-demo-app/src/Client && npx tailwindcss build"'
                }
            }
        }

        stage('Add EF Core Design Package') {
            steps {
                script {
                    sh 'docker exec dotnet6-container bash -c "cd /p3ops-demo-app/src/Server && dotnet add package Microsoft.EntityFrameworkCore.Design --version 6.0.0"'
                }
            }
        }

        stage('Build in dotnet6-container') {
            steps {
                sh 'docker exec dotnet6-container dotnet restore /p3ops-demo-app/src/Server/Server.csproj'
                sh 'docker exec dotnet6-container dotnet build /p3ops-demo-app/src/Server/Server.csproj'
            }
        }
        stage('Test SQL Server connection') {
            steps {
                script {
                    sh '''
                    docker exec dotnet6-container bash -c "
                    apt-get update && apt-get install -y iputils-ping net-tools netcat-openbsd &&
                    ping -c 4 172.17.0.3 &&
                    nc -zv 172.17.0.3 1433 || echo 'Unable to connect to SQL Server via netcat' &&
                    apt-get install -y curl apt-transport-https &&
                    curl https://packages.microsoft.com/keys/microsoft.asc | apt-key add - &&
                    curl https://packages.microsoft.com/config/ubuntu/$(lsb_release -rs)/prod.list > /etc/apt/sources.list.d/mssql-release.list &&
                    apt-get update &&
                    ACCEPT_EULA=Y apt-get install -y msodbcsql17 &&
                    ACCEPT_EULA=Y apt-get install -y mssql-tools &&
                    echo 'export PATH=\\"\\$PATH:/opt/mssql-tools/bin\\"' >> ~/.bashrc &&
                    source ~/.bashrc &&
                    sleep 5 &&
                    sqlcmd -S 172.17.0.3,1433 -U YourUsername -P YourPassword -Q \\"SELECT 1\\" || echo 'Unable to connect to SQL Server via sqlcmd'"
                    '''
                }
            }
        }

       stage('Fill database SQL Server') {
    steps {
        script {
            def toolInstalled = sh(script: 'docker exec dotnet6-container bash -c "cd /p3ops-demo-app/src/Server && dotnet tool list -g | grep dotnet-ef"', returnStatus: true)

            if (toolInstalled != 0) {
                echo 'Installing dotnet-ef tool...'
                sh 'docker exec dotnet6-container bash -c "cd /p3ops-demo-app/src/Server && dotnet tool install --global dotnet-ef --version 6.0.0"'
            } else {
                echo 'dotnet-ef tool is already installed.'
            }

            sh 'docker exec dotnet6-container bash -c "cd /p3ops-demo-app/src/Server && export PATH=$PATH:/root/.dotnet/tools"'
            sh 'docker exec dotnet6-container bash -c "cd /p3ops-demo-app/src/Server && ~/.dotnet/tools/dotnet-ef database update"'
        }
    }
}

        stage('Publish and start app') {
            steps {
                script {
                    sh 'docker exec dotnet6-container dotnet publish /p3ops-demo-app/src/Server/Server.csproj -c Release -o publish'
                    sh 'docker exec dotnet6-container bash -c "cd publish && dotnet Server.dll"'
                }
            }
        }
    }
}
