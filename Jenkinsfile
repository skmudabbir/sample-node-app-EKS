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

            REM --- Proxy off ---
            set "HTTP_PROXY="
            set "HTTPS_PROXY="
            set "NO_PROXY=localhost,127.0.0.1,::1,*.amazonaws.com,*.eks.amazonaws.com,169.254.170.2"

            REM --- Ensure PATH includes common tool dirs (no PowerShell) ---
            set "PATH=%PATH%;C:\\ProgramData\\chocolatey\\bin;C:\\Tools"

            REM --- Resolve eksctl absolute path if needed ---
            set "EKSCTL=eksctl"
            for /f "delims=" %%i in ('where eksctl 2^>NUL') do set "EKSCTL=%%i"
            if not exist "%EKSCTL%" (
              if exist "C:\\ProgramData\\chocolatey\\bin\\eksctl.exe" set "EKSCTL=C:\\ProgramData\\chocolatey\\bin\\eksctl.exe"
            )
            if not exist "%EKSCTL%" (
              if exist "C:\\Tools\\eksctl.exe" set "EKSCTL=C:\\Tools\\eksctl.exe"
            )
            "%EKSCTL%" version >NUL 2>&1 || (echo ERROR: eksctl not found for Jenkins service PATH. Add eksctl.exe to PATH or copy to C:\\Tools & exit /b 1)

            REM --- Ensure ECR repo exists ---
            aws ecr describe-repositories --repository-names %ECR_REPO% --region %AWS_DEFAULT_REGION% >NUL 2>&1
            IF ERRORLEVEL 1 (
              aws ecr create-repository --repository-name %ECR_REPO% --region %AWS_DEFAULT_REGION%
            )

            REM --- ECR login, tag, push ---
            for /f "usebackq delims=" %%p in (`aws ecr get-login-password --region %AWS_DEFAULT_REGION%`) do docker login --username AWS --password %%p %ECR_REG%
            docker tag %IMAGE_NAME%:%IMAGE_TAG% %ECR_REG%/%ECR_REPO%:%IMAGE_TAG%
            docker push %ECR_REG%/%ECR_REPO%:%IMAGE_TAG%

            REM -- Create EKS Cluster (only if missing) --
            aws eks describe-cluster --name %EKS_CLUSTER% --region %AWS_DEFAULT_REGION% >NUL 2>&1
            IF ERRORLEVEL 1 (
              echo Creating EKS cluster %EKS_CLUSTER%...
              "%EKSCTL%" create cluster ^
                --name %EKS_CLUSTER% ^
                --region %AWS_DEFAULT_REGION% ^
                --managed ^
                --nodegroup-name ng-default ^
                --nodes 2 ^
                --nodes-min 1 ^
                --nodes-max 3 ^
                --node-type t3.medium ^
                --with-oidc
              IF ERRORLEVEL 1 (echo ERROR: eksctl cluster create failed. & exit /b 1)
            ) ELSE (
              echo EKS cluster %EKS_CLUSTER% already exists. Skipping create.
            )

            REM --- Verify cluster is visible before proceeding ---
            aws eks describe-cluster --name %EKS_CLUSTER% --region %AWS_DEFAULT_REGION% >NUL 2>&1 || (echo ERROR: Cluster not ready/visible yet. & exit /b 1)

            REM --- Ensure kube dir exists before writing kubeconfig ---
            if not exist "C:\\ProgramData\\Jenkins\\.kube" mkdir "C:\\ProgramData\\Jenkins\\.kube"

            REM --- Point kubectl at EKS (writes kubeconfig file) ---
            aws eks update-kubeconfig --name %EKS_CLUSTER% --region %AWS_DEFAULT_REGION% --kubeconfig "%EKS_KUBECONFIG%"

            REM --- Sanity check context ---
            kubectl --kubeconfig="%EKS_KUBECONFIG%" config current-context || (echo ERROR: kubeconfig not usable. & exit /b 1)
            kubectl --kubeconfig="%EKS_KUBECONFIG%" cluster-info

            REM --- Apply your manifests ---
            kubectl --kubeconfig="%EKS_KUBECONFIG%" apply -f k8s-deployment.yaml
            kubectl --kubeconfig="%EKS_KUBECONFIG%" apply -f k8s-service.yaml

            REM --- Update image to ECR tag ---
            kubectl --kubeconfig="%EKS_KUBECONFIG%" set image deployment/%APP_NAME% %CONTAINER_NAME%=%ECR_REG%/%ECR_REPO%:%IMAGE_TAG%

            kubectl --kubeconfig="%EKS_KUBECONFIG%" rollout status deploy/%APP_NAME% --timeout=300s
            kubectl --kubeconfig="%EKS_KUBECONFIG%" get pods -o wide
            kubectl --kubeconfig="%EKS_KUBECONFIG%" get svc -o wide
          """
        }
      }
    }
