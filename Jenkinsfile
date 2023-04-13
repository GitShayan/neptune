//job name: neptune-api-ci
pipeline {
    agent any
    stages {
        //stage("Clone"){
        //    steps {
        //        git(
        //            url: "http://github.com/gitshayan/neptune-api"
        //            credentialsId: "github_key"
        //    )
        //    }
        //}
        stage("Load Configuration"){
            steps {
                dir("jenkins") {
                    git(
                        url: "$GIT_URL-ci-config"
                        credentialsId: "github_key"
                    )
                    load "config.groovy"
                }
                //first we clone a new repository for our configuration: api-ci-config
                //then we load the configurations in that repository we create a file and call it config.groovy
                // with the following content:
                // DOCKER_REGISTRY_ADDRESS = "192.168.12.3"
                // DOCKER_IMAGE_NAME = "neptune-api"
                // DOCKER_DB_IMAGE_NAME = "mysql:8"
            }
        }
        stage("Build"){
            steps {
                 neptuneImage = docker.build("$DOCKER_REGISTRY_ADDRESS/$DOCKER_IMAGE_NAME:latest")
                 // docker build -t 192.168.12.3/neptune:latest .
            }
        }
        stage("Test"){
            steps {
                script {
                    docker.image("$DOCKER_DB_IMAGE_NAME").withRun("--name $JOB_NAME-$BUILD_ID-mysql-test -e MYSQL_ROOT_PASSWORD=testing -e MYSQL_DATABASE=testing")
                        container ->
                        neptuneImage.inside("--name $JOB_NAME-$BUILD_ID-api-test -e NEPTUNE_API_ENV=testing -e NEPTUNE_API_DEBUG=1 -e NTEPUNE_API_TESTING=1 -e NEPTUNE_API_DATABASE_URI=mysql+pymysql://root:testing@$$JOB_NAME-$BUILD_ID-mysql-test:3306/testing --link $container.id"){
                            retry(10){
                                sh "flask app test"
                                // test application connection to database or we can use flask db upgrade
                                sleep(5)
                            }
                            sh "flask db upgrade"
                            sh "coverage run -m pytest --junit-xml=junit.xml"
                            sh "coverage html"
                            sh "coverage xml"
                            junit "junit.xml"
                            cobertura (coberturaReportFile: "coverage.xml")
                        }
                    //docker run -d --name neptune-api-ci-1-mysql-test -e MYSQL_ROOT_PASSWORD=testing -e MYSQL_DATABASE=testing mysql:8
                    //$JOB_NAME=neptune-api-ci
                    //"container" is a variable that contain ID and other information of mysql container that we create earlier 
                    // .inside used to run CMD inside of the running container, here the container that we run using variable neptuneImage that we build earlier
                    // we use two plugin in jenkins, juint to show our tests and cobertura used to show our code coverage
                }
            }
        }
        stage("Release"){
            steps {
                script {
                    neptuneImage.push()
                    gitLastCommit = sh(script: "git rev-parse HEAD", returnStdout: true).trim()
                    neptuneImage.push(gitLastCommit)
                    gitLastTag = sh(script: "git tag --point-at HEAD", returnStdout: true).trim()
                    if (gitLastTag != "") {
                        gitLastTag = gitLastTag.minus("v")
                        neptuneMajorVersion = gitLastTag.split("\\.")[0]
                        neptuneMinorVersion = gitLastTag.split("\\.")[0] + "." + gitLastTag.split("\\.")[1]
                        neptuneImage.push(neptuneMajorVersion)
                        neptuneImage.push(neptuneMinorVersion)
                    }
                    //we should at least add two tag to our image
                    //one of them is the latest git commit and another one is latest
                    //if in our git we have tag we should two more tag, "major" version and "major.minor" version
                    //gitLastTag.minus("v") it remove "v" from Tag v1.0.0 string
                }
            }
        }

    }
}
