/*
  Jenkinsfile — HAPI FHIR Observability Pipeline
  ================================================
  Stages :
    1. Checkout          — récupère le code
    2. SonarQube Analysis — analyse la qualité du code Python
    3. Quality Gate      — bloque si le score est insuffisant
    4. Stack Health      — vérifie que tous les conteneurs tournent
    5. Run Experiments   — lance le script PowerShell d'incidents
    6. Push to Prometheus — pousse les résultats CSV vers Grafana
    7. Archive           — sauvegarde tous les artefacts

  Prérequis Jenkins (plugins) :
    - Pipeline
    - SonarQube Scanner
    - AnsiColor
    - HTML Publisher
*/

pipeline {

    agent any

    environment {
        SONAR_HOST_URL   = "http://sonarqube:9000"
        PUSHGATEWAY_URL  = "http://pushgateway:9091"
        HAPI_URL         = "http://hapi-fhir-jpaserver-start:8080"
        RESULTS_CSV      = "${WORKSPACE}/results.csv"
    }

    options {
        timeout(time: 2, unit: 'HOURS')
        timestamps()
        ansiColor('xterm')
    }

    parameters {
        booleanParam(
            name: 'RUN_SONAR',
            defaultValue: true,
            description: 'Activer l\'analyse SonarQube'
        )
        booleanParam(
            name: 'RUN_EXPERIMENTS',
            defaultValue: true,
            description: 'Lancer les expériences d\'incidents'
        )
        booleanParam(
            name: 'PUSH_METRICS',
            defaultValue: true,
            description: 'Pousser les résultats vers Prometheus'
        )
    }

    stages {

        // ── 1. Checkout ──────────────────────────────────
        stage('Checkout') {
            steps {
                echo '📥 Récupération du code source...'
                checkout scm
            }
        }

        // ── 2. SonarQube Analysis ────────────────────────
        stage('SonarQube Analysis') {
            when {
                expression { return params.RUN_SONAR }
            }
            steps {
                echo '🔍 Analyse SonarQube en cours...'
                withSonarQubeEnv('SonarQube') {
                    sh """
                        sonar-scanner \
                          -Dsonar.projectKey=hapi-fhir-observability \
                          -Dsonar.projectName="HAPI FHIR Observability" \
                          -Dsonar.projectVersion=1.0 \
                          -Dsonar.sources=. \
                          -Dsonar.inclusions="**/*.py,**/*.yml,**/*.yaml" \
                          -Dsonar.exclusions="**/node_modules/**,**/__pycache__/**,**/charts/**" \
                          -Dsonar.python.version=3.11 \
                          -Dsonar.host.url=${SONAR_HOST_URL} \
                          -Dsonar.scm.disabled=true
                    """
                }
            }
        }

        // ── 3. Quality Gate ──────────────────────────────
        stage('Quality Gate') {
            when {
                expression { return params.RUN_SONAR }
            }
            steps {
                echo '🚦 Vérification du Quality Gate SonarQube...'
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false
                }
            }
        }

        // ── 4. Stack Health Check ────────────────────────
        stage('Stack Health') {
            steps {
                echo '🏥 Vérification de la stack...'
                script {
                    def services = [
                        'HAPI FHIR':   'http://localhost:9090/fhir/metadata',
                        'Prometheus':  'http://localhost:9091/-/healthy',
                        'Grafana':     'http://localhost:3000/api/health',
                        'Loki':        'http://localhost:3100/ready',
                        'SonarQube':   'http://localhost:9000/api/system/status',
                        'Pushgateway': 'http://localhost:9092/-/healthy',
                    ]
                    services.each { name, url ->
                        def code = sh(
                            script: "curl -sf -o /dev/null -w '%{http_code}' '${url}' || echo 000",
                            returnStdout: true
                        ).trim()
                        if (code == '200') {
                            echo "  ✅ ${name} → UP (HTTP ${code})"
                        } else {
                            echo "  ⚠️  ${name} → HTTP ${code} (non bloquant)"
                        }
                    }
                }
            }
        }

        // ── 5. Run Experiments ───────────────────────────
        stage('Run Experiments') {
            when {
                expression { return params.RUN_EXPERIMENTS }
            }
            steps {
                echo '🧪 Lancement des expériences d\'incidents...'
                // Sur Windows avec Jenkins, on utilise PowerShell
                powershell """
                    .\\run_experiments.ps1 `
                        -HapiFhirUrl "http://localhost:9090" `
                        -ResultsFile "${RESULTS_CSV}" `
                        -StabilizationWait 15 `
                        -MaxDetectionWait 60
                """
            }
        }

        // ── 6. Push Metrics to Prometheus ────────────────
        stage('Push Metrics') {
            when {
                expression { return params.PUSH_METRICS }
            }
            steps {
                echo '📊 Envoi des métriques vers Prometheus...'
                sh """
                    python3 csv_to_prometheus.py \
                        --csv ${RESULTS_CSV} \
                        --gateway http://localhost:9092
                """
            }
        }

        // ── 7. Archive Artifacts ─────────────────────────
        stage('Archive') {
            steps {
                echo '📦 Archivage des artefacts...'
                archiveArtifacts(
                    artifacts: 'results.csv,results_analysis.json,charts/**/*',
                    allowEmptyArchive: true,
                    fingerprint: true
                )
                // Génère le rapport d'analyse
                sh 'python3 analyze_results.py results.csv || true'
            }
        }
    }

    post {
        success {
            echo """
            ╔══════════════════════════════════════════╗
            ║  ✅ Pipeline TERMINÉ avec succès         ║
            ║  📊 Résultats dans Grafana               ║
            ║  🔍 Rapport SonarQube disponible         ║
            ╚══════════════════════════════════════════╝
            """
        }
        failure {
            echo '❌ Pipeline ÉCHOUÉ — vérifier les logs ci-dessus'
            // Tentative de nettoyage
            sh 'docker start hapi-fhir-postgres 2>/dev/null || true'
        }
        always {
            echo "Build #${BUILD_NUMBER} terminé"
        }
    }
}