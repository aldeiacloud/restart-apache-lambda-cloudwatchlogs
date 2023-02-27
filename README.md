## Automação para Reiniciar Apache via Cloudwatch Logs + AWS Lambda


---

**1- Criar uma instância EC2:**

---

**2- Adicionar as seguintes políticas na role da instância:**


Essa serve para publicar os logs no CloudWatch Logs:


```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "logs:DescribeLogStreams"
    ],
      "Resource": [
        "*"
    ]
  }
 ]
}
```

---


**3- Instalar o apache na instância:**


```
$ sudo apt install apache2
```

---


**4- Instalar o agente do Cloudwatch logs:**

  
- Para Ubuntu
    
```
sudo apt update -y
cd /tmp
curl https://s3.amazonaws.com/aws-cloudwatch/downloads/latest/awslogs-agent-setup.py -O
sudo apt install python
sudo python ./awslogs-agent-setup.py --region us-east-1
```



Na instalação, você pode adicionar quantos logs quiser, porem se quiser adicionar somente o do apache, altere a configuração padrão durante a configuração do primeiro log.

  

Outra questão é você pular as etapas de configurar uma credencial programática, porque voce ja colocou na função da instância.

---


**5- Crie uma Lambda para triggar um restart no apache2**

- Para isso, devemos criar uma rule do tipo LAMBDA, com as seguintes permissões
    


```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "logs:ListTagsLogGroup",
                "logs:GetDataProtectionPolicy",
                "ssm:SendCommand",
                "logs:GetLogRecord",
                "ec2:DescribeInstances",
                "logs:DescribeLogStreams",
                "logs:DescribeSubscriptionFilters",
                "logs:StartQuery",
                "logs:DescribeMetricFilters",
                "logs:GetLogDelivery",
                "logs:ListLogDeliveries",
                "logs:CreateLogStream",
                "logs:GetLogEvents",
                "logs:FilterLogEvents",
                "logs:DescribeQueryDefinitions",
                "logs:DescribeResourcePolicies",
                "logs:DescribeDestinations",
                "logs:DescribeQueries",
                "ec2:RebootInstances",
                "logs:DescribeLogGroups",
                "logs:Unmask",
                "logs:StopQuery",
                "logs:TestMetricFilter",
                "logs:PutLogEvents",
                "logs:CreateLogGroup",
                "logs:ListTagsForResource",
                "logs:DescribeExportTasks",
                "logs:GetQueryResults",
                "logs:GetLogGroupFields"
            ],
            "Resource": "*"
        }
    ]
}
```

  

- Vá ao painel do AWS Lambda
    
    - Crie uma nova função
        
    - Dê um nome a esta função, sugestivo
        
    - Use python 3.9
        
    - Escolha a rule que você criou com as políticas acima
        

  

- No código substitua por este aqui
    


```
import boto3
import json

def lambda_handler(event, context):
    client = boto3.client('logs')
    pattern = "erro" # Substitua por uma palavra que deseja procurar no log que deseja triggar
    log_group = "/var/log/apache2/error.log" # Substitua pelo nome do grupo de logs que deseja monitorar
    response = client.filter_log_events(logGroupName=log_group, filterPattern=pattern)
    if len(response['events']) > 0:
        ssm_client = boto3.client('ssm')
        instance_id = "i-0635fe42216b1dcd8" # Substitua pelo ID da instância que deseja reiniciar
        response = ssm_client.send_command(
            InstanceIds=[instance_id],
            DocumentName="AWS-RunShellScript",
            Parameters={'commands': ['systemctl restart apache2']}) # Substitua pelo comando que deseja executar
```

**Observações:**

Preste atenção em “pattern”, “log\_group”, “instance\_id” e “Parameters”

  

- Altere o tempo de duração para 1 min.
    

---



**7- Agora vamos configurar uma regra para o CloudWatch e triggar a lambda ao receber a palavra erro.**

  
No console da lambda que criou;

- Clique em Add Trigger,
    
- Procure por Cloudwatch Logs,
    
- Selecione seu grupo de logs que deseja filtrar,
    
- Coloque um apelido para o filtro,
    
- Adicione a palavra “erro” ou correspondente que queira filtrar para a lambda ler, depois aplique.
    

---

Pronto! Agora você criou uma regra do CloudWatch Logs que monitorará seus logs do Apache e acionará uma ação da Lambda para reiniciar o serviço sempre que receber um log com a palavra "erro".
