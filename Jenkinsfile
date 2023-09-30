// Submission Dicoding - Proyek Membangun CI/CD Pipeline dengan Jenkins
// Albertus Septian Angkuw 

def getArtifactName(){
    def NAME = sh(script: 'mvn help:evaluate -Dexpression=project.name | grep "^[^\\[]"', returnStdout: true)
    def VERSION = sh(script: 'mvn help:evaluate -Dexpression=project.version | grep "^[^\\[]"', returnStdout: true)
    return [NAME , VERSION]
}

pipeline {
    agent any
    stages {
        stage("Prepare"){
            agent {
                docker {
                    image 'maven:3.9.0'
                }
            }
            stages{
                stage('Build') {
                    steps {
                        sh 'mvn -B -DskipTests clean package'
                    }
                }
                stage('Test') { 
                    steps {
                        sh 'mvn test'
                    }
                    post {
                        always {
                            junit 'target/surefire-reports/*.xml'
                        }
                    }
                }
                stage('Deliver'){
                    
                    steps{
                        script{
                            String pidNPM
                            sh 'mvn jar:jar install:install help:evaluate -Dexpression=project.name'
                            def (NAME, VERSION) = getArtifactName()
                            pidNPM = sh(script: "java -jar target/${NAME}-${VERSION}.jar > log-server.txt 2>&1 &" + 'echo "$!"', returnStdout: true)
                            println "PID Server: $pidNPM"
                            sleep time: 1, unit: 'MINUTES'
                            // Clean Up, Aplikasi yang dijalankan
                            sh "kill $pidNPM"
                            // Menampilkan log dari aplikasi yang dijalankan
                            sh "cat log-server.txt"
                        }
                    }
                }
            }
        }
        stage('Manual Approval'){
            steps{
                input message: 'Lanjutkan ke tahap Deploy? (Click "Proceed" to continue)'
            }
        }
        stage('Deploy'){
            steps{
                script{
                    sh 'mvn jar:jar install:install help:evaluate -Dexpression=project.name'
                    
                    def (NAME, VERSION)  = getArtifactName()
                    println "Akan membuat image dengan nama: albertushub/simple-java-maven:$VERSION"
                    sh "cp Dockerfile.deploy Dockerfile"
                    sh "sed -i 's/\${app-name-gen}/{NAME}-${VERSION}.jar/g' Dockerfile"
                    withCredentials([usernamePassword(credentialsId: '6abe9262-12bf-4500-8b64-bf7181fd687b', passwordVariable: 'DOCKER_HUB_PASSWORD', usernameVariable: 'DOCKER_HUB_USERNAME')]) {
                        // Login Docker
                        sh 'docker login -u="${DOCKER_HUB_USERNAME}" -p="${DOCKER_HUB_PASSWORD}"'
                        // Build Image
                        sh "docker build -t albertushub/simple-java-maven:$VERSION ."
                        // Push Image ke Docker Hub
                        sh "docker push albertushub/simple-java-maven:$VERSION"
                        // Logout Docker
                        sh "docker logout"
                    }

                    def remote = [:]
                    // Label Remote Host
                    remote.name = "ec2-aws"
                    // IP Host
                    remote.host = "3.0.182.170"
                    remote.allowAnyHosts = true
                    // Menggunakan SSH Pipeline Steps Plugins untuk trigger docker pull pada remote server
                    withCredentials([sshUserPrivateKey(credentialsId: 'a85c8863-8a79-4370-8d84-3812897c24ea', keyFileVariable: 'identity', passphraseVariable: '', usernameVariable: 'user')]) {
                        remote.user = user
                        remote.identityFile = identity
                        stage("Remote") {
                            // sshCommand remote: remote, command: 'echo "Hello from Jenkins" > hello.txt'
                            // Pull Container yang baru dibuild dengan tag yang sudah dibuat
                            sshCommand remote: remote, command: "docker pull albertushub/simple-java-maven:$VERSION"
                            // Hentikan container app yang sudah berjalan
                            sshCommand remote: remote, command: 'docker stop simple-java-maven'
                            // Hapus container app yang lawas
                            sshCommand remote: remote, command: 'docker rm simple-java-maven'
                            // Run Container yang baru
                            sshCommand remote: remote, command: "docker run --name=simple-java-maven -d  albertushub/simple-java-maven:$VERSION"
                            
                        }
                    }
                    println("Selesai 🔥🚀 !-------")
                }
            }
        }
    }
}
