pipeline {
    agent any
    
    environment {
        AZURE_WEBAPP_NAME = 'mycheckserver-app'
        AZURE_RESOURCE_GROUP = 'TubesCloud'
    }
    
    tools {
        nodejs 'NodeJS-20'
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
                    sh 'npm ci'
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
                
                // Create unified server
                writeFile file: 'deploy/server.js', text: """
import express from 'express';
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
"""
                
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
                
                sh 'cd deploy && zip -r ../deploy.zip .'
            }
        }
        
        stage('Deploy to Azure') {
            steps {
                withCredentials([azureServicePrincipal('azure-sp-credentials')]) {
                    sh '''
                        az login --service-principal -u $AZURE_CLIENT_ID -p $AZURE_CLIENT_SECRET -t $AZURE_TENANT_ID
                        az webapp deployment source config-zip \
                            --resource-group $AZURE_RESOURCE_GROUP \
                            --name $AZURE_WEBAPP_NAME \
                            --src deploy.zip
                        az logout
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
