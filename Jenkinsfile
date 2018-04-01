node {
    stage 'Checkout'
    checkout scm

    stage 'Build'
    sh "echo Building ..."
    sh "sbt publish"
 
    stage 'Deploy'

    marathon(
        url: 'http://leader.mesos/service/marathon',
        credentialsId: 'bus-demo-ui-dcos-token'
    )
}
