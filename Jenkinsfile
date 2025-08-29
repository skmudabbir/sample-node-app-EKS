pipeline {
  agent any

  environment {
    IMAGE_NAME = 'sample-node-app-eks'
    IMAGE_TAG  = 'v1.0'
    // Local KUBECONFIG for Docker Desktop / local K8s (if you still use it)
    KUBECONFIG = 'C:\\ProgramData\\Jenkins\\.kube\\config'

    // EKS-specific
    AWS_DEFAULT_REGION = 'us-east-1'
    ACCOUNT_ID   = '291234479378'
    ECR_REPO     = 'sample-node-app-eks'
    APP_NAME     = 'sample-node-app-eks'      // must match your Deployment .metadata.name
    CONTAINER_NAME = 'sample-node-app-eks'    // must match the container name in the Deployment
    EKS_CLUSTER  = 'sample-eks'
    EKS_KUBECONFIG = 'C:\\ProgramData\\Jenkins\\.kube\\eks-config'

    // where we'll drop tools
    TOOLS_DIR = 'C:\\Tools'
    PATH = "C:\\Windows\\system32;C:\\Windows;${TOOLS_DIR};%PATH%"
  }

  stages {

    stage('Checkout Code') {
      steps {
        git branch: 'main', url: 'https://github.com/skmudabbir/sample-node-app-EKS.git'
      }
    }

    stage('Tooling Setup (eksctl/kubectl)') {
      steps {
        withEnv(["POWERSHELL_TELEMETRY_OPTOUT=1"]) {
          bat '''
            if not exist "%TOOLS_DIR%" mkdir "%TOOLS_DIR%"

            REM --- Install eksctl if missing ---
            where eksctl >NUL 2>&1
            if errorlevel 1 (
              echo Installing eksctl...
              powershell -NoProfile -ExecutionPolicy Bypass -Command ^
                "$v = (Invoke-RestMethod https://api.github.com/repos/eksctl-io/eksctl/releases/latest).tag_name; " ^
                "$url = \\"https://github.com/eksctl-io/eksctl/releases/download/$v/eksctl_Windows_amd64.zip\\"; " ^
                "$zip = \\"%TOOLS_DIR%\\\\eksctl.zip\\"; " ^
                "Invoke-WebRequest -Uri $url -OutFile $zip; " ^
                "Add-Type -AssemblyName System.IO.Compression.FileSystem; " ^
                "[IO.Compression.ZipFile]::ExtractToDirectory($zip, \\"%TOOLS_DIR%\\", $true); " ^
                "Remove-Item $zip"
            ) else (
              echo eksctl already present.
            )

            REM --- Install kubectl if missing ---
            where kubectl >NUL 2>&1
            if errorlevel 1 (
              echo Installing kubectl...
              powershell -NoProfile -ExecutionPolicy Bypass -Command ^
                "$url = \\"https://dl.k8s.io/release/stable.txt\\"; " ^
                "$ver = (Invoke-RestMethod $url).Trim(); " ^
                "$bin = \\"https://dl.k8s.io/$ver/bin/windows/amd64/kubectl.exe\\"; " ^
                "Invoke-WebRequest -Uri $bin -OutFile \\"%TOOLS_DIR%\\\\kubectl.exe\\""
            ) else (
              echo kubectl already present.
            )

            REM --- Verify aws cli present ---
            aws --version
            if errorlevel 1 (
              echo AWS CLI is required. Install AWS CLI v2 on this agent and re-run. & exit /b 1
            )

            REM --- Ensure kube dir exists ---
            if not exist "C:\\ProgramData\\Jenkins\\.kube" mkdir "C:\\ProgramData\\Jenkins\\.kube"
          '''
        }
      }
    }

    stage('Build Docker Image') {
      steps {
        script {
          docker.build("${IMAGE_NAME}:${IMAGE_TAG}")
        }
      }
    }

    stage('Push to ECR') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'aws-access',
                                          usernameVariable: 'AWS_ACCESS_KEY_ID',
                                          passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
          bat '''
            set ECR_REG=%ACCOUNT_ID%.dkr.ecr.%AWS_DEFAULT_REGION%.amazonaws.com

            REM --- ECR: ensure repo exists ---
            aws ecr describe-repositories --repository-names %ECR_REPO% --region %AWS_DEFAULT_REGION% >NUL 2>&1
            IF ERRORLEVEL 1 (
              aws ecr create-repository --repository-name %ECR_REPO% --region %AWS_DEFAULT_REGION%
            )

            REM --- Login, tag, push ---
            for /f "usebackq delims=" %%p in (`aws ecr get-login-password --region %AWS_DEFAULT_REGION%`) do docker login --username AWS --password %%p %ECR_REG%
            docker tag %IMAGE_NAME%:%IMAGE_TAG% %ECR_REG%/%ECR_REPO%:%IMAGE_TAG%
            docker push %ECR_REG%/%ECR_REPO%:%IMAGE_TAG%
          '''
        }
      }
    }

    stage('Provision EKS (idempotent)') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'aws-access',
                                          usernameVariable: 'AWS_ACCESS_KEY_ID',
                                          passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
          bat '''
            set "HTTP_PROXY="
            set "HTTPS_PROXY="
            set "NO_PROXY=localhost,127.0.0.1,::1,*.amazonaws.com,*.eks.amazonaws.com,169.254.170.2"

            REM --- Check if cluster exists ---
            aws eks describe-cluster --name %EKS_CLUSTER% --region %AWS_DEFAULT_REGION% >NUL 2>&1
            if errorlevel 1 (
              echo Creating EKS cluster %EKS_CLUSTER% in %AWS_DEFAULT_REGION%...
              eksctl create cluster ^
                --name %EKS_CLUSTER% ^
                --region %AWS_DEFAULT_REGION% ^
                --nodes 2 ^
                --nodes-min 1 ^
                --nodes-max 3 ^
                --managed ^
                --nodegroup-name ng-default ^
                --with-oidc
            ) else (
              echo EKS cluster %EKS_CLUSTER% already exists. Skipping create.
              REM --- Ensure a managed nodegroup exists (create if missing) ---
              eksctl get nodegroup --cluster %EKS_CLUSTER% --region %AWS_DEFAULT_REGION% | findstr /i "ng-default" >NUL 2>&1
              if errorlevel 1 (
                echo Creating default managed nodegroup...
                eksctl create nodegroup ^
                  --cluster %EKS_CLUSTER% ^
                  --region %AWS_DEFAULT_REGION% ^
                  --name ng-default ^
                  --nodes 2 ^
                  --nodes-min 1 ^
                  --nodes-max 3 ^
                  --managed
              ) else (
                echo Managed nodegroup present.
              )
            )

            REM --- Write kubeconfig to desired path for Jenkins service account ---
            aws eks update-kubeconfig --name %EKS_CLUSTER% --region %AWS_DEFAULT_REGION% --kubeconfig "%EKS_KUBECONFIG%"

            kubectl --kubeconfig="%EKS_KUBECONFIG%" config current-context
            kubectl --kubeconfig="%EKS_KUBECONFIG%" cluster-info
          '''
        }
      }
    }

    stage('Deploy to EKS') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'aws-access',
                                          usernameVariable: 'AWS_ACCESS_KEY_ID',
                                          passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
          bat '''
            set ECR_REG=%ACCOUNT_ID%.dkr.ecr.%AWS_DEFAULT_REGION%.amazonaws.com

            set "HTTP_PROXY="
            set "HTTPS_PROXY="
            set "NO_PROXY=localhost,127.0.0.1,::1,*.amazonaws.com,*.eks.amazonaws.com,169.254.170.2"

            REM --- Apply manifests ---
            kubectl --kubeconfig="%EKS_KUBECONFIG%" apply -f k8s-deployment.yaml
            kubectl --kubeconfig="%EKS_KUBECONFIG%" apply -f k8s-service.yaml

            REM --- Point image at ECR without changing YAML ---
            kubectl --kubeconfig="%EKS_KUBECONFIG%" set image deployment/%APP_NAME% %CONTAINER_NAME%=%ECR_REG%/%ECR_REPO%:%IMAGE_TAG% --record

            kubectl --kubeconfig="%EKS_KUBECONFIG%" rollout status deploy/%APP_NAME% --timeout=300s
            kubectl --kubeconfig="%EKS_KUBECONFIG%" get pods -o wide
            kubectl --kubeconfig="%EKS_KUBECONFIG%" get svc -o wide
          '''
        }
      }
    }
  }

  post {
    success { echo 'Pipeline completed successfully!' }
    failure { echo 'Pipeline failed. Check console output for details.' }
  }
}
