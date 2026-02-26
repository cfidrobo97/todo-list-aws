pipeline {
  agent any
  options {
    timestamps()
    disableConcurrentBuilds()
  }

  environment {
    AWS_REGION   = "us-east-1"
    AWS_DEFAULT_REGION = "us-east-1"
    STACK_NAME   = "production-todo-list-aws"
    STAGE_ENV   = "production"
    OUTPUT_KEY   = "BaseUrlApi"     
    PY_VENV      = ".venv"
  }

  stages {

    stage('Get Code') {
      steps {
        withCredentials([string(credentialsId: 'github_pat', variable: 'GITHUB_TOKEN')]) {
          sh '''
            set -e
            rm -rf repo && mkdir repo
            cd repo
            git clone -b master https://$GITHUB_TOKEN@github.com/cfidrobo97/todo-list-aws.git .
        
          '''
        }
      }
    }


    stage('Deploy') {
      steps {
        sh '''
          set -e
          cd repo
          sam build
          sam validate --region "${AWS_REGION}"

          sam deploy \
            --stack-name "${STACK_NAME}" \
            --region "${AWS_REGION}" \
            --capabilities CAPABILITY_IAM \
            --no-confirm-changeset \
            --no-fail-on-empty-changeset \
            --parameter-overrides Stage=${STAGE_ENV} \
            --resolve-s3
        '''
      }
    }

    stage('Rest Test') {
      steps {
        sh '''
          set -e
          cd repo

          # Instala y preparar entorno virtual
          python3 -m venv ${PY_VENV}
          ${PY_VENV}/bin/pip install -U pip
          ${PY_VENV}/bin/pip install -U pytest requests

          # Obtener Base URL desde CloudFormation Outputs
          BASE_URL=$(aws cloudformation describe-stacks \
            --stack-name "${STACK_NAME}" \
            --region "${AWS_REGION}" \
            --query "Stacks[0].Outputs[?OutputKey=='${OUTPUT_KEY}'].OutputValue" \
            --output text)

          echo "BASE_URL=${BASE_URL}"
          test -n "$BASE_URL"  

          # Export para que lo use pytest
          export BASE_URL

          # Ejecutar pruebas de integración
          ${PY_VENV}/bin/python -m pytest -s -k "listtodos" test/integration/todoApiTest.py
        '''
      }
    }
  }
}