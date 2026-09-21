# Checkliste — AWS Academy Lab starten, EC2/ECR/ECS einrichten, GitHub Secrets setzen

Für den kompletten Durchlauf der Pipeline (`test` → `build` → `deploy` → `docker` → `deploy-ecs`)
in einer frischen AWS-Academy-Learner-Lab-Session. Reihenfolge von oben nach unten abarbeiten.

---

## 0. Lab starten

- [ ] AWS Academy: **Start Lab**, warten bis der Punkt neben "AWS" grün ist
- [ ] *AWS Details → AWS CLI* öffnen — von dort kommen gleich `AWS_ACCESS_KEY_ID`,
      `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`
- [ ] AWS-Console öffnen, Region oben rechts auf **us-east-1 (N. Virginia)** stellen
      (Learner Lab ist darauf fixiert — `task-definition.json` und `deploy.yml` gehen
      ebenfalls von `us-east-1` aus)

> ⚠️ Diese Zugangsdaten laufen mit der Lab-Sitzung ab (spätestens nach 4h, oder
> beim nächsten Neustart der Session). Schritt 4 (Secrets setzen) muss dann erneut
> durchgeführt werden.

---

## 1. EC2-Instanz für das Frontend (EX-01)

- [ ] EC2 → **Launch instance**
  - Name: z. B. `biztrips-frontend`
  - AMI: Amazon Linux 2023 (Login-User `ec2-user`) **oder** Ubuntu (Login-User `ubuntu`)
  - Instance-Type: `t3.micro` (Learner-Lab-Limit)
  - Key Pair: **vockey** (Private Key `labsuser.pem` unter *AWS Details → Download PEM*)
  - Security Group: Inbound `SSH (22)` und `HTTP (80)`, Quelle `0.0.0.0/0`
- [ ] Public IPv4 DNS notieren (z. B. `ec2-3-91-12-34.compute-1.amazonaws.com`)
- [ ] Erreichbarkeit **vor** dem ersten Pipeline-Run testen:
  ```bash
  nc -z -v <public-dns> 22
  ssh -i labsuser.pem <user>@<public-dns> true && echo "ssh ok"
  ```
- [ ] nginx + rsync auf der Instanz installieren, `/var/www/biztrips` anlegen
      (Details: [EX-01](EX-01-deploy-AWS-EC2.md) Schritt 2–3, alternativ
      [EX-00](EX-00-Vorbereitungen-AWS-Linux.md) für Amazon-Linux-spezifische Befehle —
      **Achtung:** EX-00 nutzt `dnf`/Amazon Linux, das README `apt`/Ubuntu; je nach
      gewählter AMI den passenden Befehlssatz verwenden)
- [ ] `http://<public-dns>/` liefert eine Testseite, **bevor** die Pipeline angefasst wird

---

## 2. ECR-Repository + ECS-Cluster/Service (EX-03)

Alles in **us-east-1**, Ressourcennamen so wählen, dass sie zu den Namen im
Workflow passen (`ECR_REPOSITORY: biztrips`, `cluster: biztrips-cluster`,
`service: biztrips-service`, `family: biztrips` in `task-definition.json`):

- [ ] `aws ecr create-repository --repository-name biztrips --region us-east-1`
- [ ] `aws ecs create-cluster --cluster-name biztrips-cluster --region us-east-1`
- [ ] Application Load Balancer + Target Group (Typ `ip`, Port 80, Health-Check `/`)
      anlegen ([EX-03](EX-03-deploy-AWS-ECS.md) Schritt 4)
- [ ] Security Group der Fargate-Tasks: Port 80 nur von der ALB-Security-Group
- [ ] `task-definition.json` im Repo-Root anpassen:
  - [ ] `executionRoleArn` → `arn:aws:iam::<eure-account-id>:role/LabRole`
        (**nicht** `ecsTaskExecutionRole` — die lässt sich im Learner Lab nicht anlegen)
  - [ ] `<eure-account-id>` in der `image`-URI ersetzen (wird beim Deploy zwar
        überschrieben, sollte aber konsistent sein)
- [ ] Task Definition registrieren: `aws ecs register-task-definition --cli-input-json file://task-definition.json --region us-east-1`
- [ ] `aws ecs create-service ...` mit `--launch-type FARGATE`, `--desired-count 2`,
      Subnets/Security-Group/Target-Group aus den vorigen Schritten
      ([EX-03](EX-03-deploy-AWS-ECS.md) Schritt 6)

---

## 3. GitHub Secrets & Variables setzen

### 3a. Environment `production` (nur für den `deploy`-Job, EC2)

*Settings → Environments → production → Add secret* (Environment ggf. zuerst anlegen):

| Secret | Wert |
| --- | --- |
| `EC2_HOST` | Public DNS aus Schritt 1 |
| `EC2_USER` | `ec2-user` (Amazon Linux) oder `ubuntu` (Ubuntu-AMI) |
| `EC2_SSH_KEY` | vollständiger Inhalt von `labsuser.pem` (inkl. `BEGIN`/`END`-Zeilen) |

```bash
gh secret set EC2_HOST --env production --body "<public-dns>"
gh secret set EC2_USER --env production --body "<ec2-user|ubuntu>"
gh secret set EC2_SSH_KEY --env production < labsuser.pem
```

### 3b. Repository Secrets (Docker Hub, `docker`-Job)

| Secret | Wert |
| --- | --- |
| `DOCKERHUB_USERNAME` | euer Docker-Hub-Benutzername |
| `DOCKERHUB_TOKEN` | Access Token aus *Docker Hub → Account Settings → Security* (kein Passwort!) |

```bash
gh secret set DOCKERHUB_USERNAME --body "<dockerhub-user>"
gh secret set DOCKERHUB_TOKEN --body "<access-token>"
```

### 3c. Repository Secrets (AWS Academy Learner Lab, `deploy-ecs`-Job)

Werte aus *AWS Details → AWS CLI* im Lab (Schritt 0) — **läuft mit der Session ab,
muss bei jeder neuen Lab-Sitzung wiederholt werden**:

| Secret | Wert |
| --- | --- |
| `AWS_ACCESS_KEY_ID` | `aws_access_key_id` aus dem Lab |
| `AWS_SECRET_ACCESS_KEY` | `aws_secret_access_key` aus dem Lab |
| `AWS_SESSION_TOKEN` | `aws_session_token` aus dem Lab |

```bash
gh secret set AWS_ACCESS_KEY_ID --body "<...>"
gh secret set AWS_SECRET_ACCESS_KEY --body "<...>"
gh secret set AWS_SESSION_TOKEN --body "<...>"
```

### 3d. Repository Variables (`build`- und `docker`-Job)

*Settings → Secrets and variables → Actions → Variables*:

| Variable | Wert |
| --- | --- |
| `VITE_API_BASE_URL` | z. B. `http://<ec2-host-des-backends>:3001/` |
| `VITE_IMGS` | `items` (Default, falls nicht gesetzt) |

---

## 4. Kontrolle vor dem Push

- [ ] `gh secret list` und `gh secret list --env production` zeigen alle sechs Secrets
- [ ] `gh variable list` zeigt `VITE_API_BASE_URL`, `VITE_IMGS`
- [ ] `task-definition.json` enthält keine Platzhalter-Account-ID mehr
- [ ] ECR-Repo, Cluster, Service, ALB stehen und sind in **us-east-1**

Danach: Push auf `main` (oder `workflow_dispatch`) auslöst alle fünf Jobs;
`deploy`, `docker`, `deploy-ecs` laufen nur bei Push auf `main`, nicht bei Pull Requests.
