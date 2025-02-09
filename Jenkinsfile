
pipeline{
    agent{
        label 'workernode-1'
    }

    parameters{
        choice (name: 'buildOnly',
                choices: 'no\nyes',
                description: "Build the Application only")
        choice (name: 'dockerformat',
                choices: 'no\nyes',
                description: "format docker")
        choice (name: 'dockerPush',
                choices: 'no\nyes',
                description: "this will push to registry")
        choice (name: 'deployToDev',
                choices: 'no\nyes',
                description: "Deploy app to Dev only ")
        choice (name: 'deployToStg',
                choices: 'no\nyes',
                description: "Deploy app to Stage only ")
        choice (name: 'deployToProd',
                choices: 'no\nyes',
                description: "Deploy app to Prod only ")
    }

    tools{
        maven 'mvn-3.8.8'
        jdk 'jdk-17'
    }
    environment {
        Application_Name = "user"
        Pom_Version = readMavenPom().getVersion()
        Pom_Packaging = readMavenPom().getPackaging()
        Docker_Hub = "docker.io/aadil08"
        Docker_Creds = credentials('docker_creds')
        //Docker_Host_ip
    }
    stages{
        stage ('Build'){
            when {
                anyOf{
                    expression {
                        params.buildOnly == "yes"
                    }
                }
            }

            // This will takee care of building the application
            steps{
                script{
                    buildApp().call()
                }
            }
        }

        /*stage ('Unit Tests'){
            steps {
                echo "Executing Unit Tests for ${env.Application_Name} App"
                sh 'mvn test'
            }
        } 
        */

        stage ('Docker Format'){
             when {
                anyOf{
                    expression {
                        params.dockerformat == "yes"
                    }
                }
            }
            // This is to format artifact
            steps{
                //install Pipeline Utility to use readMavenPOM
                //i27-eureka-0.0.1-SNAPSHOT.jar →> eureka-build number-branch name.jar
                echo "The actual format is ${env.Application_Name}-${env.Pom_Version}.${env.Pom_Packaging}"

                 // Expected ureka-buildnumber-branchname.jar to get build number in Jenkins goto Pipeline Syntax> Global Variables Reference

                 echo "Custom Format is ${env.Application_Name}-${BUILD_NUMBER}-${BRANCH_NAME}.${env.Pom_Packaging}"

            }
        }

        stage ('Docker Build & Push') {
             when {
                anyOf{
                    expression {
                        params.dockerPush == "yes"
                    }
                }
            }
            steps{
                script{
               dockerBuildandPush().call()
                }
            }
        }

        stage ('Deploy to Dev'){
           when {
                anyOf{
                    expression {
                        params.deployToDev == "yes"
                    }
                }
            }
            steps {
                script{
                    imageValidation().call()
                    dockerDeploy('dev', '5232', '8232' ).call()
                }
            }
        }

        stage ("Deploy to Stage"){
          when {
                anyOf{
                    expression {
                        params.deployToStg == "yes"
                    }
                }
            }
            steps {
                script{
                    imageValidation().call()
                    dockerDeploy('stg', '7232', '8232' ).call()
                }
            }
        }

        stage ("Deploy to Prod With approval"){
            when {
                allOf {
                    anyOf {
                        expression {
                            params.deployToProd == "yes"
                        }
                    }

                    anyOf {
                        expression {
                            branch 'release/*'
                        }
                    }
                }
            }
            steps {
                script{
                    timeout(time:300, unit: 'SECONDS'){
                    input message: "Deploying to ${env.Application_Name} to Prod", ok: 'yes', submitter: 'jack'
                    }
                    imageValidation().call()
                    dockerDeploy('prd', '8232', '8232' ).call()
                }
            }
        }       
        
    }
}

// Create a Method to Deploy our application into various environments

 def dockerDeploy(envDeploy, hostPort, contPort){
     return {
        // for every env, what will change: Application Name, HostPort, ContainerPort, Container Name, Environment Name
            echo "*************************  Deploy to ${envDeploy}  *****************************"
            script{
            sh "docker pull  ${env.Docker_Hub}/${env.Application_Name}:${GIT_COMMIT}"
            try{
               //Stop the container
                sh "docker stop ${env.Application_Name}-${envDeploy}"
                //remove the container
                sh "docker rm ${env.Application_Name}-${envDeploy}"
            } catch(err){
                echo "Error Caught: $err"
              }
            // create the container
            echo "*************************  Running the  ${envDeploy} Container  *****************************"
            sh "docker run -d -p ${hostPort}:${contPort} --name ${env.Application_Name}-${envDeploy} ${env.Docker_Hub}/${env.Application_Name}:${GIT_COMMIT}"
            }
     }
 }

def imageValidation(){
    return{
        println ("Pulling the docker image")
        try{
            sh "docker pull  ${env.Docker_Hub}/${env.Application_Name}:${GIT_COMMIT}"
        }
        catch (Exception e){
            println("Docker image with this tag doesnt exist, so creating the image")
            buildApp().call()
            dockerBuildandPush().call()
        }
    }
}

def buildApp(){
    return {
        echo "Building ${env.Application_Name} Application"
        // build using maven
        sh 'mvn clean package -DskipTests=true'
        archiveArtifacts artifacts: 'target/*.jar'
    }
}

def dockerBuildandPush(){
    return{
        echo "Starting Docker Build "
        echo "Copy the jar to the folder where Docker file is present"
        sh "cp ${WORKSPACE}/target/i27-${env.Application_Name}-${env.Pom_Version}.${env.Pom_Packaging} ./.cicd/"
        echo "********************* Building Docker Image ********************"
        sh "docker build --force-rm  --no-cache --build-arg JAR_SRC=i27-${env.Application_Name}-${env.Pom_Version}.${env.Pom_Packaging}  -t ${env.Docker_Hub}/${env.Application_Name}:${GIT_COMMIT} ./.cicd"
        echo "********************* Login to Docker Repo ********************"
        sh "docker login -u ${Docker_Creds_USR} -p ${Docker_Creds_PSW}"
        echo "********************* Docker Push ********************"
        sh "docker push ${env.Docker_Hub}/${env.Application_Name}:${GIT_COMMIT}"
    }
}

 // --------------------------------------------- For Reference -----------------------------------------

        //  stage ('Deploy to Dev'){
        //     steps {
        //        // echo "*************************  Deploy to Dev  *****************************"
        //         //withCredentials([usernamePassword(credentialsId: 'ali_dock-vm', passwordVariable: 'password', usernameVariable: 'username')]) {
        //         //With this block Slave will connect to docker server using ssh and will execute the commands we want.
        //         // Connect to the Docker Server from Jenkin machine. Credentials for user ali are stored in Jenkin  using the above. The user ali is root user in Docker machine which we created.
        //         // When we execute this, jenking master initiates jenkin slave. Jenkin slave will connect to Docker Vm using the cred and execuste
                
                
        //         // As we dont want to hardcode ip address, we can keep public ip addrs as "Environment variables" under Manage Jenkins>System>Global Properties
        //         //sshpass -p password ssh -o StrictHostKeyChecking=no username@ipaddress command_to_run. 
        //         // Command: docker run -d -p hp:cp --name containername image:tagname ( container port 8761 is always same in all different environment for Eureka application)
        //         // docker run -d -p 5761:8761 --name ${env.Application_Name}-dev ${env.Docker_Hub}/${env.Application_Name}:${GIT_COMMIT}
        //       //  sh "sshpass -p  ${password} ssh -o StrictHostKeyChecking=no ${username}@${docker_server_ip} docker run -d -p 5761:8761 --name ${env.Application_Name}-dev ${env.Docker_Hub}/${env.Application_Name}:${GIT_COMMIT}"
        //         //}
        //         echo "*************************  Deploy to Dev  *****************************"
        //         script{
        //             // This block is written because if the container is running already and run the jenkin file it was throwing error because the container name already exists so we are writing the below
        //             // to make sure we catch any exception in try catch.

        //             // pull the image 
        //           sh "docker pull  ${env.Docker_Hub}/${env.Application_Name}:${GIT_COMMIT}"

        //           try{
        //             //Stop the container
        //             sh "docker stop ${env.Application_Name}-dev"

        //             //remove the container
        //             sh "docker rm ${env.Application_Name}-dev"

        //           } catch(err){
        //             echo "Error Caught: $err"
        //           }
        //           // create the container
        //         echo "*************************  Running the Dev Container  *****************************"
        //         sh "docker run -d -p 5761:8761 --name ${env.Application_Name}-dev ${env.Docker_Hub}/${env.Application_Name}:${GIT_COMMIT}"
        //         }
        //     }
        // }