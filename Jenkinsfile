pipeline {
    agent any

    environment {
        SONAR_TOKEN = credentials('sonar-token')
    }

    stages {

        stage('Identity Validation') {
            steps {
                sh '''
                    set -e
                    GIT_USERNAME=$(git log -1 --pretty=format:"%an")
                    GIT_EMAIL=$(git log -1 --pretty=format:"%ae")
                    echo "Committer: $GIT_USERNAME <$GIT_EMAIL>"

                    if ! echo "$GIT_USERNAME" | grep -qP "\\."; then
                        echo "[FAIL] Username must contain a dot."
                        exit 1
                    fi

                    if ! echo "$GIT_EMAIL" | grep -qP "@optimumdataanalytics\\.com$"; then
                        echo "[FAIL] Email must end with @optimumdataanalytics.com"
                        exit 1
                    fi

                    echo "[PASS] Identity validation passed."
                '''
            }
        }

        stage('PyLint Code Quality') {
            steps {
                sh '''
                    set -e
                    pip3 install pylint --quiet --break-system-packages
                    PYTHON_FILES=$(find . -name "*.py" -not -path "./.git/*" -not -path "./venv/*")

                    if [ -z "$PYTHON_FILES" ]; then
                        echo "No Python files found. Skipping."
                        exit 0
                    fi

                    PYLINT_OUTPUT=$(pylint $PYTHON_FILES 2>&1 || true)
                    echo "$PYLINT_OUTPUT"

                    SCORE=$(echo "$PYLINT_OUTPUT" | grep -oP "(?<=rated at )\\d+\\.?\\d*" | head -1)

                    if [ -z "$SCORE" ]; then
                        echo "[FAIL] Could not extract PyLint score."
                        exit 1
                    fi

                    echo "PyLint Score: $SCORE / 10"
                    PASS=$(awk -v s="$SCORE" "BEGIN { print (s >= 6) ? 1 : 0 }")
                    [ "$PASS" -eq 1 ] || { echo "[FAIL] Score $SCORE is below 6."; exit 1; }
                    echo "[PASS] PyLint check passed."
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '/opt/sonar-scanner/bin/sonar-scanner -Dsonar.login=$SONAR_TOKEN'
                }
            }
        }
    }

    post {
        success {
            echo 'All checks passed. Ready for review and merge.'
        }
        failure {
            echo 'Pipeline failed. Check the stage logs above.'
        }
    }
}
