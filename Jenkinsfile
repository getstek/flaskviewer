job('Frontend build and push') {
    scm {
        git('https://github.com/getstek/flaskviewer.git') {  node -> // is hudson.plugins.git.GitSCM
            node / gitConfigName('DSL User')
            node / gitConfigEmail('jenkins-dsl@manbearpig.run')
        }
    }
    triggers {
        scm('H/5 * * * *')
    }
    steps {
        dockerBuildAndPublish {
            repositoryName('localhost:5000/flask-dsl')
            tag('${GIT_REVISION,length=9}')
            forcePull(false)
            forceTag(false)
            createFingerprints(false)
            skipDecorate()
        }
    }
}
