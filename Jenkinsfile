pipeline {
  agent any

  environment {
    IMAGE_NAME = 'sample-node-app-eks'
    IMAGE_TAG  = 'v1.0'
    KUBECONFIG = 'C:\\ProgramData\\Jenkins\\.kube\\config'
  }

  stages {
    stage('Checkout Code') {
      steps {
        git branch: 'main', url: 'https://github.com/skmudabbir/sample-node-app-EKS.git'
      }
    }

    stage('Build Docker Image') {
      steps {
        script {
          docker.build("${IMAGE_NAME}:${IMAGE_TAG}")
        }
      }
    }

    stage('Deploy to Kubernetes') {
      steps {
        bat '''
          REM --- Make sure kubectl bypasses any proxies in Jenkins env ---
          set "HTTP_PROXY="
          set "HTTPS_PROXY="
          set "NO_PROXY=localhost,127.0.0.1,::1,kubernetes.docker.internal,*.local,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16"

          kubectl apply -f k8s-deployment.yaml
          kubectl apply -f k8s-service.yaml
          kubectl scale deployment node-app-deployment --replicas=3
        '''
      }
    }

    stage('Deploy to EKS') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'aws-access',
                                          usernameVariable: 'AWS_ACCESS_KEY_ID',
                                          passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
          bat """
            REM --- Config ---
            set AWS_DEFAULT_REGION=us-east-1
            set ACCOUNT_ID=291234479378
            set ECR_REPO=sample-node-app-eks
            set APP_NAME=sample-node-app-eks
            set CONTAINER_NAME=sample-node-app-eks
            set IMAGE_TAG=v1.0
            set EKS_CLUSTER=sample-eks
            set ECR_REG=%ACCOUNT_ID%.dkr.ecr.%AWS_DEFAULT_REGION%.amazonaws.com
            set EKS_KUBECONFIG=C:\\ProgramData\\Jenkins\\.kube\\eks-config

            REM --- Check Proxy issue ---
            set "HTTP_PROXY="
            set "HTTPS_PROXY="
            set "NO_PROXY=localhost,127.0.0.1,::1,*.amazonaws.com,*.eks.amazonaws.com,169.254.170.2"

            REM --- Ensure ECR repo exists  ---
            aws ecr describe-repositories --repository-names %ECR_REPO% --region %AWS_DEFAULT_REGION% >NUL 2>&1
            IF ERRORLEVEL 1 (
              aws ecr create-repository --repository-name %ECR_REPO% --region %AWS_DEFAULT_REGION%
            )

            REM --- ECR login, tag, push ---
            for /f "usebackq delims=" %%p in (`aws ecr get-login-password --region %AWS_DEFAULT_REGION%`) do docker login --username AWS --password %%p %ECR_REG%
            docker tag %IMAGE_NAME%:%IMAGE_TAG% %ECR_REG%/%ECR_REPO%:%IMAGE_TAG%
            docker push %ECR_REG%/%ECR_REPO%:%IMAGE_TAG%

            REM -- Create EKS Cluster (minimal change: use eksctl, only if missing) --
            aws eks describe-cluster --name %EKS_CLUSTER% --region %AWS_DEFAULT_REGION% >NUL 2>&1
            IF ERRORLEVEL 1 (
              echo Creating EKS cluster %EKS_CLUSTER%...
              eksctl create cluster ^
                --name %EKS_CLUSTER% ^
                --region %AWS_DEFAULT_REGION% ^
                --managed ^
                --nodegroup-name ng-default ^
                --nodes 2 ^
                --nodes-min 1 ^
                --nodes-max 3 ^
                --node-type t3.medium ^
                --with-oidc
            ) ELSE (
              echo EKS cluster %EKS_CLUSTER% already exists. Skipping create.
            )

            REM --- Ensure kube dir exists before writing kubeconfig (minimal add) ---
            if not exist "C:\\ProgramData\\Jenkins\\.kube" mkdir "C:\\ProgramData\\Jenkins\\.kube"

            REM --- Point kubectl at EKS (writes kubeconfig file for the Jenkins service) ---
            aws eks update-kubeconfig --name %EKS_CLUSTER% --region %AWS_DEFAULT_REGION% --kubeconfig "%EKS_KUBECONFIG%"

            kubectl --kubeconfig="%EKS_KUBECONFIG%" config current-context
            kubectl --kubeconfig="%EKS_KUBECONFIG%" cluster-info

            REM --- Apply your manifests (namespace optional) ---
            kubectl --kubeconfig="%EKS_KUBECONFIG%" apply -f k8s-deployment.yaml
            kubectl --kubeconfig="%EKS_KUBECONFIG%" apply -f k8s-service.yaml

            REM --- Point the Deployment to the pushed ECR image (keeps YAML static) ---
            kubectl --kubeconfig="%EKS_KUBECONFIG%" set image deployment/%APP_NAME% %CONTAINER_NAME%=%ECR_REG%/%ECR_REPO%:%IMAGE_TAG% --record

            kubectl --kubeconfig="%EKS_KUBECONFIG%" rollout status deploy/%APP_NAME% --timeout=180s
            kubectl --kubeconfig="%EKS_KUBECONFIG%" get pods -o wide
            kubectl --kubeconfig="%EKS_KUBECONFIG%" get svc -o wide
          """
        }
      }
    }
  }

  post {
    success {
      echo 'Pipeline completed successfully!'
    }
  }
}
