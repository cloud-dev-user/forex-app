node {
    // Parameters for user input
    properties([
        parameters([
            string(name: 'AMOUNT', defaultValue: '1000', description: 'Amount in INR'),
            choice(name: 'CURRENCY', choices: ['USD', 'EUR', 'AED', 'GBP', 'JPY'], description: 'Target currency')
        ])
    ])

    stage('Checkout') {
        checkout scm
    }
    
    stage('Setup Maven') {
        def mvnHome = tool name: 'Default', type: 'hudson.tasks.Maven$MavenInstallation'
        env.PATH = "${mvnHome}/bin:${env.PATH}"
        echo "Using Maven at: ${mvnHome}"
    }

    stage('Build') {
        sh 'mvn clean compile'
    }

    stage('Test & Coverage') {
        sh 'mvn test jacoco:report'
        junit 'target/surefire-reports/*.xml'
        publishHTML(target: [
            reportDir: 'target/site/jacoco',
            reportFiles: 'index.html',
            reportName: 'Jacoco Coverage Report'
        ])
    }

    stage('Package') {
        sh 'mvn package -DskipTests'
        archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
    }

    stage('Run Forex App') {
        sh "java -cp target/forex-app-1.0.0.jar com.example.forex.ForexConverter rates.csv ${params.AMOUNT} ${params.CURRENCY}"
    }

    stage('Prepare Workspace Path') {
        script {
            def agentName = 'agent01'
            def node = Jenkins.instance.getNode(agentName)
            if (node) {
                def base = node.remoteFS.endsWith('/') ? node.remoteFS : node.remoteFS + '/'
                def agentWorkspace = "${base}workspace/${env.JOB_NAME}/"
                env.AGENT_WORKSPACE = agentWorkspace
            } else {
                error("Agent '${agentName}' not found")
            }
        }
    }

    stage('Copy files to agent01 workspace') {
        withCredentials([sshUserPrivateKey(credentialsId: 'ec2-user', keyFileVariable: 'KEY', usernameVariable: 'USERNAME')]) {
            withCredentials([string(credentialsId: 'aws_server', variable: 'AGENT_HOST')]) {
                sh """
                    scp -i ${KEY} -o StrictHostKeyChecking=no rates.csv Dockerfile target/forex-app-1.0.0.jar ec2-user@${AGENT_HOST}:${AGENT_WORKSPACE}
                """
            }
        }
    }

    stage('Docker Build & Run') {
        node('agent01') {
            sh '''
              sudo docker build -t forex-app .
              sudo docker run --rm forex-app ${AMOUNT} ${CURRENCY}
            '''
        }
    }
}
