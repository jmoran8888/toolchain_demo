node() {
    checkout scm
    stage('Scrub Pipeline') {
        // important to cleanup pipeline artifacts
        sh 'kubectl delete deployments --ignore-not-found=true springboot-demo'
        // remove load balancer service
        sh 'kubectl delete services --ignore-not-found=true springboot-demo'
        sh 'rm -f /home/project/build/libs/*' // -f to avoid failure if dir is empty
    }
    /* Docker pipeline plugin installed in Jenkins container */
    docker.image('gradle:latest').inside('--network=toolchain_demo_tc-net') {
        def tasks = [:]

        stage('Build') {
            // commented lines for inspection
            // sh  'gradle buid --scan' // find build dependencies including transitive and build report
            // sh ' gradle dependencies' // just list the dependencies, no report
            sh 'gradle bootJar -p /home/project'
        }

        tasks['unittest'] = {
            stage('Unit Test') {
                // all unit test tasks
                sh 'gradle test -p /home/project'
            }
        }
        tasks['bddtest'] = {
            stage('BDD Test') {
                sh 'gradle cucumberTest -p /home/project'
            }
        }
        tasks['integrationtest'] = {
            stage('Integration Test') {
                sh 'gradle integrationTest -p /home/project'
            }
        }
        tasks['codeanalysis'] = {
            stage('Code Analysis') {
                // run sonarqube
                sh 'gradle sonarqube -p /home/project'
            }
        }

        parallel tasks

        stage('Publish Package') {
            sh 'gradle publish -p /home/project'
        }
        stage('Retrieve App') {
            // Make the output directory.
            sh "mkdir -p output"
            // AppArtifactWs = "${env.WORKSPACE}"
            sh 'curl -u admin:admin123 -X GET "http://package-repo:8081/repository/maven-snapshots/org/ahl/springbootdemo/spring-boot-demo/0.0.1-SNAPSHOT/spring-boot-demo-0.0.1-20181203.175519-7.jar" --output ./output/app.jar'      
            
            stash name: 'app', includes: 'output/*'
        }
    }

    stage('App Image Build') {
        // NOTE: When building a different application, simply change the build-arg to point to the replacement jar
        // Run unstash within app directory
        sh "echo 'dir on app'"
        dir("app") {
            unstash "app"
        }

        // putting a comment here to see if I can push this update...
        sh "echo contents of app dir..."
        sh "ls -la ${pwd()}/output/*"

        def custom_app_image = docker.build("springboot", "--build-arg JAR_FILE=./output/app.jar -f spring-boot-demo/Dockerfile .") 
    }

    stage ('Deploy To Kube') {
        sh 'kubectl create -f /kube/deploy/app_set/sb-demo-deployment.yaml'
    }
    stage('Configure Kube Load Balancer') {
        sh 'kubectl create -f /kube/deploy/app_set/sb-demo-service.yaml'
    }
}