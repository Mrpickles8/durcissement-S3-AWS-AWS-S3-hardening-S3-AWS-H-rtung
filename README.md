# durcissement-S3-AWS-AWS-S3-hardening-S3-AWS-H-rtung
AWS project to harden a S3 bucket

> 🌍 **Select your language / Choisissez votre langue**

[🇫🇷 Français](#français) ·
[🇬🇧 English](#english) ·
[🇩🇪 Deutsch](#deutsch) ·

---

## Français

## Objectif

Ce projet démontre la capacité à repérer des mauvaises configurations de sécurité courantes sur les buckets S3 et ce qui les entoures AWS, puis à les corriger de façon reproductible avec Terraform. La démarche suit une logique avant / après : on déploie d'abord une configuration non sécurisée, puis on la durcit contrôle par contrôle.


## Compétences acquises

- Maîtrise pratique de Terraform et de l'approche Infrastructure as Code.
- Compréhension approfondie du durcissement des services AWS (S3, IAM, KMS, VPC).
- Application du principe du moindre privilège dans la conception des rôles IAM.
- Mise en place de la traçabilité et de la surveillance (CloudTrail, VPC Flow Logs).
- Développement d'un réflexe « défenseur » : repérer une mauvaise configuration et la corriger.

## Outils utilisés
- Terraform (Infrastructure as Code)
- AWS : VPC, S3, IAM, KMS, CloudTrail, VPC Flow Logs
- Chiffrement : AWS KMS (clé gérée, avec rotation automatique)
## Étapes

Ref 1 : État « avant » — stockage S3 non protégé

Le bucket initial, sans blocage d'accès public ni chiffrement configuré. C'est la situation vulnérable de départ.
```hcl
code :
resource "aws_s3_bucket" "faible" {
  bucket = "projet-a-bucket-faible-potaufeu"
}
```
![Bucket S3 sans chiffrement](images/C4.png)

Ref 2 : État « avant » — pare-feu ouvert à tout internet

Le security group autorise l'ensemble du trafic entrant depuis 0.0.0.0/0, sur tous les ports. Configuration à corriger en priorité.

```hcl
resource "aws_security_group" "faible" {
  name        = "sg-trop-ouvert"
  description = "Volontairement non securise"

  ingress {
    from_port   = 0
    to_port     = 65535
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

![Bucket S3 sans chiffrement](images/C5.png)

Ref 3 : État « après » — blocage de l'accès public S3
```hcl
Le même bucket, avec « Block public access » activé et le chiffrement au repos via une clé KMS gérée.
resource "aws_s3_bucket" "securise" {
  bucket = "projet-a-bucket-securise-potaufeu-plus-sur"
}

# Dans un premier temps on bloque TOUT accès public
resource "aws_s3_bucket_public_access_block" "securise" {
  bucket                  = aws_s3_bucket.securise.id
  block_public_acls       = true
  block_public_policy      = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Ensuite on va chiffrer avec notre clé KMS
resource "aws_s3_bucket_server_side_encryption_configuration" "securise" {
  bucket = aws_s3_bucket.securise.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.principale.arn
    }
  }
}
```
![Bucket S3 sans chiffrement](images/C6r.png)
![Bucket S3 sans chiffrement](images/C7.png)
Ref 4 : État « après » — pare-feu resserré

Le security group durci : accès limité à un seul port et une seule adresse IP source, selon le principe du moindre accès.
```hcl
code:
#ici on crée le VPC puis on le segment en sous réseau.
resource "aws_vpc" "principal" {
  cidr_block = "10.0.0.0/16"
  tags       = { Name = "vpc-projet-a" }
}

resource "aws_subnet" "prive" {
  vpc_id     = aws_vpc.principal.id
  cidr_block = "10.0.1.0/24"
  tags       = { Name = "subnet-prive" }
}

# Puis on endurcie le groupe de sécurité\Pare-feu en laissant uniquement une connexion SSH depuis une certaine IP
resource "aws_security_group" "securise" {
  name   = "sg-durci"
  vpc_id = aws_vpc.principal.id

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["203.0.113.10/32"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

![Bucket S3 sans chiffrement](images/C9r.png)

Ref 5 : Chiffrement — clé KMS avec rotation

Clé KMS gérée par le client, rotation automatique activée, utilisée pour chiffrer le stockage.
```hcl
code:
resource "aws_kms_key" "principale" {
  description             = "Cle de chiffrement projet A"
  deletion_window_in_days = 7
  enable_key_rotation     = true
}

resource "aws_kms_alias" "principale" {
  name          = "alias/projet-a"
  target_key_id = aws_kms_key.principale.key_id
}
```
![Bucket S3 sans chiffrement](images/jsp%202r.png)
Ref 6 : Traçabilité — CloudTrail actif

CloudTrail configuré en multi-région avec validation de l'intégrité des journaux : chaque action sur le compte est enregistrée.
```hcl
code:
data "aws_caller_identity" "moi" {}

# --- Bucket de logs ---
#Ici nous allons créer un nouveau bucket pour stocker les logs en créant une policy pour que CloudTrail puisse avoir accèss et utiliser ces logs via les deux actions "s3:GetBucketAcl" et "s3:PutObject". 
resource "aws_s3_bucket" "logs" {
  bucket = "projet-a-logs-CHANGE-MOI-12345"
}
resource "aws_s3_bucket_public_access_block" "logs" {
  bucket                  = aws_s3_bucket.logs.id
  block_public_acls       = true
  block_public_policy      = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
resource "aws_s3_bucket_policy" "logs" {
  bucket = aws_s3_bucket.logs.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      { Effect = "Allow", Principal = { Service = "cloudtrail.amazonaws.com" },
        Action = "s3:GetBucketAcl", Resource = aws_s3_bucket.logs.arn },
      { Effect = "Allow", Principal = { Service = "cloudtrail.amazonaws.com" },
        Action = "s3:PutObject",
        Resource = "${aws_s3_bucket.logs.arn}/AWSLogs/${data.aws_caller_identity.moi.account_id}/*" }
    ]
  })
}

# --- CloudTrail ---
#Maintenant nous configurons CloudTrail pour qu'il soit multi-régionnal et qu'Il y ait une validation des logs avant analyse.
resource "aws_cloudtrail" "principal" {
  name                       = "projet-a-trail"
  s3_bucket_name             = aws_s3_bucket.logs.id
  is_multi_region_trail      = true
  enable_log_file_validation = true
  depends_on                 = [aws_s3_bucket_policy.logs]
```
![Bucket S3 sans chiffrement](images/C10r.png)

Ref 7 : Surveillance réseau — VPC Flow Logs

Les VPC Flow Logs capturent le trafic réseau du VPC, complétant la visibilité offerte par CloudTrail.
```hcl
# --- VPC Flow Logs ---
#Ici nous avons la suite du code précédent pour activer Le traqueur de fluxs de AWS a travers le vpc.
resource "aws_flow_log" "vpc" {
  vpc_id               = aws_vpc.principal.id
  traffic_type         = "ALL"
  log_destination      = aws_s3_bucket.logs.arn
  log_destination_type = "s3"
}
```
![Bucket S3 sans chiffrement](images/C11.png)

## Note

GuardDuty n'a pas été inclus dans cette version : son activation nécessite une éligibilité de compte qui n'était pas disponible au moment de la réalisation. La détection repose ici sur CloudTrail et les VPC Flow Logs. Le projet est un exercice de durcissement, et non une architecture de production (ni haute disponibilité, ni gestion multi-comptes).

[⬆️ Retour au menu](#nom-du-projet)

---

## English

## Objective

This project demonstrates the ability to identify common security misconfigurations in S3 bucket ans its surrounding AWS and then fix them in a reproducible manner using Terraform. The approach follows a “before and after” logic: first, an insecure configuration is deployed, and then it is hardened one control at a time.

## Skills Learned
[Bullet Points - Remove this afterwards]

Advanced understanding of SIEM concepts and practical application.
Proficiency in analyzing and interpreting network logs.
Ability to generate and recognize attack signatures and patterns.
Enhanced knowledge of network protocols and security vulnerabilities.
Development of critical thinking and problem-solving skills in cybersecurity.

## Tools Used

- Terraform (Infrastructure as Code)
- AWS: VPC, S3, IAM, KMS, CloudTrail, VPC Flow Logs
- Encryption: AWS KMS (managed key, with automatic rotation)

## Steps

drag & drop screenshots here or use imgur and reference them using imgsrc

Every screenshot should have some text explaining what the screenshot is about.

Example below.
[⬆️ Back to menu](#nom-du-projet)

---

## Deutsch

## Ziel

Dieses Projekt zeigt, wie sich häufige Sicherheitskonfigurationsfehler bei S3-Buckets und im umgebenden AWS-Umfeld erkennen und anschließend mit Terraform reproduzierbar beheben lassen. Der Ansatz folgt einer Vorher-Nachher-Logik: Zunächst wird eine unsichere Konfiguration bereitgestellt, die dann Schritt für Schritt abgesichert wird.

## Erworbene Fähigkeiten

## Verwendete Werkzeuge
- Terraform (Infrastructure as Code)
- AWS: VPC, S3, IAM, KMS, CloudTrail, VPC Flow Logs
- Verschlüsselung: AWS KMS (verwalteter Schlüssel mit automatischer Rotation)

## Schritte



[⬆️ Zurück zum Menü](#nom-du-projet)
