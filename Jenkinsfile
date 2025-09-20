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
        // Determine the agent workspace path dynamically
        def agentName = 'agent01'
        def agentWorkspace = null
        
        def nodeObj = Jenkins.instance.getNode(agentName)
        if (nodeObj) {
            def base = nodeObj.remoteFS.endsWith('/') ? nodeObj.remoteFS : nodeObj.remoteFS + '/'
            agentWorkspace = "${base}workspace/${env.JOB_NAME}/"
            // Store to env for safe use in sh
            env.AGENT_WORKSPACE = agentWorkspace
            echo "Workspace path for ${agentName}: ${env.AGENT_WORKSPACE}"
        } else {
            error("Agent '${agentName}' not found")
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
            sh """
                sudo docker build -t forex-app .
                sudo docker run --rm forex-app ${params.AMOUNT} ${params.CURRENCY}
            """
        }
    }
}
