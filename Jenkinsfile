pipeline {
    // 스테이지 별로 다른 거
    // 우리는 Node가 한 개이므로 any
    agent any

    triggers {
        // 크롬 문법으로 적은 젠킨스 실행 주기
        // 3분 마다 한 번
        pollSCM('*/3 * * * *')
    }

    // 시스템 환경 변수로 들어가게 되는 것들
    environment {
      // AWS 를 사용하기 위한 Access Key 등록
      AWS_ACCESS_KEY_ID = credentials('awsAccessKeyId') // awsAccessKeyId는 Jenkins의 Global credentials secret text로 등록되어 있음
      AWS_SECRET_ACCESS_KEY = credentials('awsSecretAccessKey') // awsSecretAccessKey는 Jenkins의 Global credentials secret text로 등록되어 있음
      AWS_DEFAULT_REGION = 'ap-northeast-2' // 서울
      HOME = '.' // Avoid npm root owned
    }
    
    // 젠킨스의 각 스테이지: Prepare, Only for production, Deploy Frontend, Lint Backent, Test Backent, Build Backend, Deploy Backend
    stages {       
        
        // 레포지토리를 다운로드 받음
        stage('Prepare') {
            agent any
            
            steps {
                echo 'Clonning Repository'

                git url: 'https://github.com/lsmman/Jenkins-experiment.git',
                    branch: 'main',
                    credentialsId: 'githubToken'
            }

            post {
                // If Maven was able to run the tests, even if some of the test
                // failed, record the test results and archive the jar file.
                success {
                    echo 'Successfully Cloned Repository'
                }

                always {
                  echo "i tried..."
                }

                cleanup {
                  echo "after all other post condition" // 잘 마치면 메일이나 슬랙 같은 로그도 추가 
                }
            }
        }
        
//         stage('Only for production') {
//             when { // 언제 실행할 지 설정 가능
//                 branch 'production'
//                 environment name: 'APP_ENV', value: 'prod' // APP_ENV가 prod일 때만 실행
//                 anyOf {
//                     environment name: 'DEPLOY_TO', value: 'production'
//                     environment name: 'DEPLOY_TO', value: 'staging'
//                 }
//             }
//         }
                    
        
        // aws s3 에 파일을 올림
        stage('Deploy Frontend') {
          steps {
            echo 'Deploying Frontend'
            // 프론트엔드 디렉토리의 정적파일들을 S3 에 올림, 이 전에 반드시 EC2 instance credential profile 을 등록해야함.
            dir ('./website'){ // root directory에서 우리가 옮기고픈 website로 cd 명령어
                sh '''
                aws s3 sync ./ s3://lsh-s3-jenkins
                '''
            }
          }

          post {
              // If Maven was able to run the tests, even if some of the test
              // failed, record the test results and archive the jar file.
              success {
                  echo 'Successfully Cloned Repository'

                  mail  to: 'lsmman5211@gmail.com',
                        subject: "Deploy Frontend Success",
                        body: "Successfully deployed frontend!"

              }

              failure {
                  echo 'I failed :('

                  mail  to: 'lsmman5211@gmail.com',
                        subject: "Failed Pipelinee",
                        body: "Something is wrong with deploy frontend"
              }
          }
        }
        
        stage('Lint Backend') {
            // Docker plugin and Docker Pipeline 두개를 깔아야 사용가능!
            agent { // Docker를 가지고 일을 할 거다.                                                                                                                                                                                                                                                                                                                                                                                                                                   
              docker {
                image 'node:latest'
              }
            }
            
            steps {
              dir ('./server'){
                  sh '''
                  npm install&&
                  npm run lint // lint script
                  '''
              }
            }
        }
        
        stage('Test Backend') {
          agent {
            docker {
              image 'node:latest'
            }
          }
          steps {
            echo 'Test Backend'

            dir ('./server'){
                sh '''
                npm install
                npm run test
                '''
            }
          }
        }
        
        stage('Bulid Backend') {
          agent any
          steps {
            echo 'Build Backend'

            dir ('./server'){
                sh """
                docker build . -t server --build-arg env=${PROD}
                """
            }
          }

          post {
            failure {
              error 'This pipeline stops here...'
            }
          }
        }
        
        stage('Deploy Backend') {
          agent any

          steps {
            echo 'Build Backend'

            dir ('./server'){
                sh '''
                // docker rm -f $(docker ps -aq) // 도커 돌고 있는 거 다 꺼버리는 것
                docker run -p 80:80 -d server // ECS를 쓰고 있으면 ECS 업데이트를 해준다.
                '''
            }
          }

          post {
            success {
              mail  to: 'lsmman5211@gmail.com',
                    subject: "Deploy Success",
                    body: "Successfully deployed!"
                  
            }
          }
        }
    }
}
