pipeline {
    agent {
        docker {
            image 'node:20-alpine'
            args '-u root'
        }
    }
    
    environment {
        AZURE_WEBAPP_NAME = 'mycheckserver-app'
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Install Dependencies') {
            steps {
                sh 'npm ci'
                dir('backend') {
                    sh 'npm ci --ignore-scripts'
                }
            }
        }
        
        stage('Build Frontend') {
            steps {
                sh 'npm run build'
            }
        }
        
        stage('Prepare Deployment') {
            steps {
                sh '''
                    rm -rf deploy deploy.zip
                    mkdir -p deploy/public
                    cp -r dist/* deploy/public/
                    cp -r backend/* deploy/
                    rm -rf deploy/node_modules deploy/data.db*
                '''
                
                writeFile file: 'deploy/server.js', text: '''import express from 'express';
import path from 'path';
import { fileURLToPath } from 'url';
import cors from 'cors';
import dotenv from 'dotenv';

const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);

dotenv.config();

const app = express();
const PORT = process.env.PORT || 8080;

app.use(cors());
app.use(express.json());
app.use(express.static(path.join(__dirname, 'public')));

import authRoutes from './routes/auth.js';
import serverRoutes from './routes/servers.js';
import notificationRoutes from './routes/notifications.js';
import billingRoutes from './routes/billing.js';

app.use('/api/auth', authRoutes);
app.use('/api/servers', serverRoutes);
app.use('/api/notifications', notificationRoutes);
app.use('/api/billing', billingRoutes);

app.get('*', (req, res) => {
  res.sendFile(path.join(__dirname, 'public', 'index.html'));
});

app.listen(PORT, () => console.log('Server running on port ' + PORT));
'''
                
                writeFile file: 'deploy/package.json', text: '''{
  "name": "mycheckserver",
  "version": "1.0.0",
  "type": "module",
  "main": "server.js",
  "scripts": { "start": "node server.js" },
  "dependencies": {
    "bcryptjs": "^2.4.3",
    "better-sqlite3": "^9.6.0",
    "cors": "^2.8.5",
    "dotenv": "^16.4.5",
    "express": "^4.21.0",
    "jsonwebtoken": "^9.0.2",
    "midtrans-client": "^1.3.1",
    "node-cron": "^3.0.3",
    "nodemailer": "^6.9.14",
    "uuid": "^10.0.0"
  },
  "engines": { "node": ">=18.0.0" }
}'''
                
                sh 'apk add --no-cache zip curl'
                sh 'cd deploy && zip -r ../deploy.zip .'
            }
        }
        
        stage('Deploy to Azure') {
            steps {
                withCredentials([string(credentialsId: 'azure-publish-profile', variable: 'PUBLISH_PROFILE')]) {
                    sh '''
                        USER=$(echo "$PUBLISH_PROFILE" | grep -oP 'userName="\\K[^"]+' | head -1)
                        PASS=$(echo "$PUBLISH_PROFILE" | grep -oP 'userPWD="\\K[^"]+' | head -1)
                        
                        curl -X POST \
                            -u "$USER:$PASS" \
                            --data-binary @deploy.zip \
                            "https://${AZURE_WEBAPP_NAME}.scm.azurewebsites.net/api/zipdeploy"
                    '''
                }
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
        success {
            echo "Deployed to https://${AZURE_WEBAPP_NAME}.azurewebsites.net"
        }
    }
}
