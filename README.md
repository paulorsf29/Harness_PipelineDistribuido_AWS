TODOS OS COMANDOS USADOS NO PROJETO — DO INICIO AO FIM
Pipeline Distribuido de Analise de Documentos


================================================================
PARTE 1 — CONFIGURACAO DE CREDENCIAIS (repetir a cada sessao)
================================================================

export AWS_ACCESS_KEY_ID=COLE_AQUI
export AWS_SECRET_ACCESS_KEY=COLE_AQUI
export AWS_SESSION_TOKEN=COLE_AQUI
export AWS_DEFAULT_REGION=us-east-1

# Verificar credenciais
aws sts get-caller-identity


================================================================
PARTE 2 — CRIACAO DOS RECURSOS AWS (feito uma unica vez)
================================================================

--- S3: Buckets de entrada e saida ---

aws s3 mb s3://pdfs-input-lab-paulo --region us-east-1
aws s3 mb s3://pdfs-output-lab-paulo --region us-east-1
aws s3 ls


--- SQS: Dead Letter Queue (criar primeiro) ---

aws sqs create-queue \
  --queue-name chunks-dlq \
  --region us-east-1

# Pegar ARN da DLQ
aws sqs get-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/609331005234/chunks-dlq \
  --attribute-names QueueArn


--- SQS: Fila principal com DLQ vinculada ---

aws sqs create-queue \
  --queue-name chunks-queue \
  --attributes '{
    "VisibilityTimeout": "120",
    "RedrivePolicy": "{\"deadLetterTargetArn\":\"arn:aws:sqs:us-east-1:609331005234:chunks-dlq\",\"maxReceiveCount\":\"3\"}"
  }' \
  --region us-east-1


--- DynamoDB: Tabela de resultados parciais ---

aws dynamodb create-table \
  --table-name chunks-results \
  --attribute-definitions \
    AttributeName=doc_id,AttributeType=S \
    AttributeName=chunk_id,AttributeType=S \
  --key-schema \
    AttributeName=doc_id,KeyType=HASH \
    AttributeName=chunk_id,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST \
  --region us-east-1

# Verificar
aws dynamodb list-tables


--- IAM: Role e Instance Profile para os workers EC2 ---

cat > /tmp/trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "ec2.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

aws iam create-role \
  --role-name WorkerRole \
  --assume-role-policy-document file:///tmp/trust-policy.json

aws iam attach-role-policy \
  --role-name WorkerRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonSQSFullAccess

aws iam attach-role-policy \
  --role-name WorkerRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess

aws iam attach-role-policy \
  --role-name WorkerRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonDynamoDBFullAccess

aws iam attach-role-policy \
  --role-name WorkerRole \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchFullAccess

aws iam create-instance-profile \
  --instance-profile-name WorkerProfile

aws iam add-role-to-instance-profile \
  --instance-profile-name WorkerProfile \
  --role-name WorkerRole


--- EC2: Security Group, AMI e instancias ---

# Pegar VPC padrao
VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query "Vpcs[0].VpcId" --output text)

# Criar Security Group
SG_ID=$(aws ec2 create-security-group \
  --group-name workers-sg \
  --description "Workers security group" \
  --vpc-id $VPC_ID \
  --query "GroupId" --output text)

# Liberar SSH
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp --port 22 --cidr 0.0.0.0/0

# Pegar AMI mais recente Amazon Linux 2023
AMI_ID=$(aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=al2023-ami-*-x86_64" \
             "Name=state,Values=available" \
  --query "sort_by(Images, &CreationDate)[-1].ImageId" \
  --output text)

# Lancar Worker 1
INSTANCE_1=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t3.medium \
  --key-name vockey \
  --security-group-ids $SG_ID \
  --iam-instance-profile Name=WorkerProfile \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=worker-1}]' \
  --query "Instances[0].InstanceId" --output text)

# Lancar Worker 2
INSTANCE_2=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t3.medium \
  --key-name vockey \
  --security-group-ids $SG_ID \
  --iam-instance-profile Name=WorkerProfile \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=worker-2}]' \
  --query "Instances[0].InstanceId" --output text)

# Aguardar instancias ficarem prontas
aws ec2 wait instance-running --instance-ids $INSTANCE_1 $INSTANCE_2

# Ver IPs
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query "Reservations[].Instances[].[Tags[?Key=='Name'].Value|[0],PublicIpAddress]" \
  --output table


================================================================
PARTE 3 — INSTALACAO DO OLLAMA NAS INSTANCIAS EC2
================================================================

# SSH na instancia (repetir para Worker 1 e Worker 2)
ssh -i /tmp/lab-key.pem -o StrictHostKeyChecking=no ec2-user@IP_WORKER

# Dentro da instancia:
sudo dnf update -y
sudo dnf install -y python3 python3-pip git
curl -fsSL https://ollama.com/install.sh | sh
sudo systemctl start ollama
sleep 5
ollama pull llama3.2:1b
echo "✅ Ollama pronto!"

# Testar modelo
ollama run llama3.2:1b "O que é MapReduce? Responda em uma frase."
# Digite /bye para sair

exit


================================================================
PARTE 4 — CRIACAO DO CODIGO DO PROJETO
================================================================

# Criar estrutura de diretorios
mkdir -p ~/pipeline/{prompts,src}

# Criar prompts versionados
cat > ~/pipeline/prompts/system_summarize.txt << 'EOF'
Voce e um assistente especializado em analise de documentos tecnicos.
Recebera um trecho (chunk) de um documento maior.
Produza um resumo conciso deste trecho em ate 3 frases.
Responda APENAS com o resumo, sem introducoes.
Versao: 1.0
EOF

cat > ~/pipeline/prompts/system_entities.txt << 'EOF'
Voce e um extrator de entidades nomeadas.
Dado o trecho abaixo, extraia entidades nos tipos:
PESSOA, ORGANIZACAO, LOCAL, DATA, CONCEITO_TECNICO.
Responda APENAS com JSON valido no formato:
{"entidades": [{"texto": "...", "tipo": "..."}]}
Se nao houver entidades, responda: {"entidades": []}
Versao: 1.0
EOF

cat > ~/pipeline/prompts/system_classify.txt << 'EOF'
Classifique o trecho abaixo em exatamente uma categoria:
INTRODUCAO, METODOLOGIA, RESULTADOS, DISCUSSAO, CONCLUSAO, REFERENCIAS, OUTRO
Responda APENAS com a categoria, sem texto adicional.
Versao: 1.0
EOF

# Copiar codigo para as instancias EC2
scp -i /tmp/lab-key.pem -o StrictHostKeyChecking=no \
  -r ~/pipeline ec2-user@IP_WORKER_1:~/

scp -i /tmp/lab-key.pem -o StrictHostKeyChecking=no \
  -r ~/pipeline ec2-user@IP_WORKER_2:~/

# Instalar dependencias nos workers
ssh -i /tmp/lab-key.pem -o StrictHostKeyChecking=no ec2-user@IP_WORKER_1 \
  "pip3 install boto3 requests pypdf --quiet && echo 'Worker 1 deps OK'"

ssh -i /tmp/lab-key.pem -o StrictHostKeyChecking=no ec2-user@IP_WORKER_2 \
  "pip3 install boto3 requests pypdf --quiet && echo 'Worker 2 deps OK'"


================================================================
PARTE 5 — EXECUCAO DO PIPELINE (repetir a cada teste)
================================================================

--- Verificar IPs atuais (mudam a cada reinicio do lab) ---

aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query "Reservations[].Instances[].[Tags[?Key=='Name'].Value|[0],PublicIpAddress]" \
  --output table


--- Restaurar chave SSH ---

rm -f /tmp/lab-key.pem
cat > /tmp/lab-key.pem << 'EOF'
-----BEGIN RSA PRIVATE KEY-----
[CONTEUDO DA CHAVE vockey — baixar do botao "Download PEM" no lab]
-----END RSA PRIVATE KEY-----
EOF
chmod 400 /tmp/lab-key.pem


--- Iniciar Worker 1 (substitua IP1) ---

ssh -i /tmp/lab-key.pem -o StrictHostKeyChecking=no ec2-user@IP1 "cat > /tmp/run_worker.sh << SCRIPT
#!/bin/bash
export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
export AWS_SESSION_TOKEN=$AWS_SESSION_TOKEN
export AWS_DEFAULT_REGION=us-east-1
export WORKER_ID=worker-1
cd ~/pipeline/src
python3 -u worker.py
SCRIPT
nohup bash /tmp/run_worker.sh > /tmp/worker1.log 2>&1 & echo 'Worker 1 OK'"


--- Iniciar Worker 2 (substitua IP2) ---

ssh -i /tmp/lab-key.pem -o StrictHostKeyChecking=no ec2-user@IP2 "cat > /tmp/run_worker.sh << SCRIPT
#!/bin/bash
export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
export AWS_SESSION_TOKEN=$AWS_SESSION_TOKEN
export AWS_DEFAULT_REGION=us-east-1
export WORKER_ID=worker-2
cd ~/pipeline/src
python3 -u worker.py
SCRIPT
nohup bash /tmp/run_worker.sh > /tmp/worker2.log 2>&1 & echo 'Worker 2 OK'"


--- Subir PDF para o S3 ---

# Baixar PDF do arXiv (paper Transformer — Attention is All You Need)
curl -L -A "Mozilla/5.0" "https://arxiv.org/pdf/1706.03762v5" -o /tmp/transformer.pdf
du -h /tmp/transformer.pdf

# Enviar para S3
aws s3 cp /tmp/transformer.pdf s3://pdfs-input-lab-paulo/
aws s3 ls s3://pdfs-input-lab-paulo/


--- Processar PDFs (roda no Worker 1 — Python 3.9) ---

# Substitua IP1 pelo IP atual do worker-1
ssh -i /tmp/lab-key.pem -o StrictHostKeyChecking=no ec2-user@IP1 "export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID; export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY; export AWS_SESSION_TOKEN=$AWS_SESSION_TOKEN; export AWS_DEFAULT_REGION=us-east-1; cd ~/pipeline/src && python3 process_pdf.py"

# ANOTE o doc_id que aparecer!


--- Monitorar workers processando em paralelo ---

# Worker 1
ssh -i /tmp/lab-key.pem -o StrictHostKeyChecking=no ec2-user@IP1 "tail -15 /tmp/worker1.log"

# Worker 2
ssh -i /tmp/lab-key.pem -o StrictHostKeyChecking=no ec2-user@IP2 "tail -15 /tmp/worker2.log"

# Ver logs ao vivo (Ctrl+C para parar)
ssh -i /tmp/lab-key.pem -o StrictHostKeyChecking=no ec2-user@IP1 "tail -f /tmp/worker1.log"


--- Reducer: agregar resultados (substitua DOC_ID) ---

cd ~/pipeline/src
export DOC_ID=COLE_O_DOC_ID_AQUI
python3 reducer.py


--- Ver resultado final ---

# Listar resultados no S3
aws s3 ls s3://pdfs-output-lab-paulo/results/

# Baixar e visualizar JSON
aws s3 cp s3://pdfs-output-lab-paulo/results/SEU_DOC_ID.json /tmp/result.json
cat /tmp/result.json


================================================================
PARTE 6 — VERIFICACOES E MONITORAMENTO
================================================================

--- CloudWatch: metricas de latencia ---

aws cloudwatch get-metric-statistics \
  --namespace PipelineAnalise \
  --metric-name ChunkLatencyMs \
  --dimensions Name=WorkerID,Value=worker-1 \
  --start-time $(date -u -d '2 hours ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 3600 \
  --statistics Average,Maximum,Minimum \
  --region us-east-1


--- SQS: verificar mensagens na DLQ (tolerancia a falhas) ---

aws sqs get-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/609331005234/chunks-dlq \
  --attribute-names ApproximateNumberOfMessages \
  --region us-east-1


--- DynamoDB: ver chunks processados ---

aws dynamodb scan \
  --table-name chunks-results \
  --select COUNT \
  --region us-east-1


--- S3: listar todos os resultados gerados ---

aws s3 ls s3://pdfs-output-lab-paulo/results/
aws s3 ls s3://pdfs-input-lab-paulo/


================================================================
PARTE 7 — RECRIACAO DE RECURSOS (se o lab resetar tudo)
================================================================

# Verificar o que ainda existe
echo "=== S3 ===" && aws s3 ls
echo "=== SQS ===" && aws sqs list-queues
echo "=== DynamoDB ===" && aws dynamodb list-tables
echo "=== EC2 ===" && aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query "Reservations[].Instances[].[Tags[?Key=='Name'].Value|[0],State.Name]" \
  --output table

# Se as instancias sumirem, recriar:
SG_ID=$(aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=workers-sg" \
  --query "SecurityGroups[0].GroupId" --output text)

AMI_ID=$(aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=al2023-ami-*-x86_64" "Name=state,Values=available" \
  --query "sort_by(Images, &CreationDate)[-1].ImageId" --output text)

aws ec2 run-instances \
  --image-id $AMI_ID --instance-type t3.medium \
  --key-name vockey --security-group-ids $SG_ID \
  --iam-instance-profile Name=WorkerProfile \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=worker-1}]' \
  --count 1

aws ec2 run-instances \
  --image-id $AMI_ID --instance-type t3.medium \
  --key-name vockey --security-group-ids $SG_ID \
  --iam-instance-profile Name=WorkerProfile \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=worker-2}]' \
  --count 1


================================================================
RECURSOS CRIADOS — REFERENCIA RAPIDA
================================================================

Conta AWS:       609331005234
Regiao:          us-east-1
S3 entrada:      pdfs-input-lab-paulo
S3 saida:        pdfs-output-lab-paulo
SQS principal:   chunks-queue
SQS DLQ:         chunks-dlq
DynamoDB:        chunks-results
IAM Role:        WorkerRole
IAM Profile:     WorkerProfile
Security Group:  workers-sg
Key Pair:        vockey
Instancias:      worker-1, worker-2 (t3.medium)
Modelo LLM:      llama3.2:1b via Ollama
CloudWatch NS:   PipelineAnalise


================================================================
RESULTADOS OBTIDOS NOS TESTES
================================================================

Teste 1 — texto de exemplo (texto_exemplo.txt):
  doc_id: 4404e1e1-b208-43d7-8524-130e1a585e86
  chunks: 4
  workers: worker-1 e worker-2
  latencia media: 67.590ms

Teste 2 — PDF real (transformer.pdf, arXiv 1706.03762):
  doc_id: c944fb00-4876-4012-9e1f-6f23b71256dd
  PDF: 2.2MB, 15 paginas, 39.300 caracteres
  chunks: 31 gerados, 25 processados
  workers: worker-1 e worker-2
  latencia media: 198.300ms
  retry automatico: 2 tentativas por timeout do Ollama

================================================================
FIM DO ARQUIVO
================================================================
