# Jenkins Practice Project

Idi oka simple Python calculator app + pytest tests + Jenkinsfile — Jenkins
pipeline ni **నిజంగా run చేసి చూడటానికి** ready-made project.

## Project Structure

```
jenkins-practice-project/
├── app.py             # Simple calculator functions (add, subtract, multiply, divide)
├── test_app.py         # PyTest tests for app.py (5 tests)
├── requirements.txt    # Python dependency: pytest
└── Jenkinsfile          # Pipeline definition - Build, Test, Package stages
```

## Step 1: Jenkins ready గా ఉందో చూడు

Jenkins ఇంకా install చేయకపోతే, ఇది run చేయి:
```bash
docker run -d -p 8080:8080 -p 50000:50000 --name jenkins-server -v jenkins_home:/var/jenkins_home jenkins/jenkins:lts
```

Password తీసుకోవడానికి:
```bash
docker exec jenkins-server cat /var/jenkins_home/secrets/initialAdminPassword
```

Browser లో `http://localhost:8080` open చేసి setup పూర్తి చేయి (ఇది ఇప్పటికే
చేసి ఉంటే ఈ Step skip చేయవచ్చు).

## Step 2: Jenkins container లో Python ఉందో చూడు

Jenkins default image లో Python ఉండదు — దాన్ని install చేయాలి, లేకపోతే
`pip install` step fail అవుతుంది.

```bash
docker exec -u root jenkins-server bash -c "apt-get update && apt-get install -y python3 python3-pip && ln -sf /usr/bin/python3 /usr/bin/python && ln -sf /usr/bin/pip3 /usr/bin/pip"
```

Verify చేయడానికి:
```bash
docker exec jenkins-server python3 --version
```

## Step 3: Pipeline Job Create చేయి

1. Jenkins dashboard → **"New Item"**
2. Name: `calculator-pipeline`, type: **Pipeline** → **OK**
3. కింద **Pipeline** section కి scroll చేయి
4. **Definition**: "Pipeline script" select చేయి
5. **ఇక్కడ ఒక చిన్న సమస్య ఉంది** — Jenkins container కి నీ లోకల్ folder
   నేరుగా కనిపించదు. కాబట్టి 2 options:

### Option A (సులభం, ఇప్పుడు జల్దీ try చేయడానికి)
`Jenkinsfile` లో ఉన్న content ని కాపీ చేసి, **"Pipeline script"** box లో
paste చేయి. కానీ `app.py`/`test_app.py`/`requirements.txt` files
Jenkins కి కనిపించవు కాబట్టి, ఈ pipeline కేవలం stages ఎలా run
అవుతాయో చూపించడానికే పనికొస్తుంది (`pip install`/`pytest` commands
"file not found" error ఇస్తాయి). Idi అర్థం చేసుకోవడానికి పర్వాలేదు.

### Option B (అసలైన ప్రాజెక్ట్ లాగా run చేయాలంటే — recommended)
నీ ప్రాజెక్ట్ ఫోల్డర్ ని Jenkins container లోపలికి **volume mount**
చేయాలి. కొత్తగా Jenkins ని ఇలా run చేయి (పాత container తీసేసి):

```bash
docker stop jenkins-server
docker rm jenkins-server
docker run -d -p 8080:8080 -p 50000:50000 --name jenkins-server ^
  -v jenkins_home:/var/jenkins_home ^
  -v C:\Users\sowji\Downloads\jenkins-practice-project:/workspace-project ^
  jenkins/jenkins:lts
```

(పైన `C:\Users\sowji\Downloads\jenkins-practice-project` బదులు, నువ్వు ఈ
ప్రాజెక్ట్ ఎక్కడ extract చేశావో ఆ exact path పెట్టు. Windows PowerShell లో
ఒకే లైన్ లో రాయాలంటే line-break `^` బదులు మొత్తం ఒకే లైన్ లో పేస్ట్
చేయవచ్చు.)

తర్వాత Jenkinsfile లో ప్రతి `sh` command ముందు `cd /workspace-project &&`
add చేయాలి:

```groovy
sh 'cd /workspace-project && pip install -r requirements.txt'
sh 'cd /workspace-project && pytest -v'
```

(ఈ project లో ఇప్పటికే ఉన్న `Jenkinsfile` ని ఈ మార్పుతో అప్‌డేట్ చేసి,
అప్పుడు "Pipeline script" box లో paste చేయి.)

### Option C (అత్యుత్తమం, నిజమైన production style — Git వాడి)
1. ఈ folder ని GitHub కి push చేయి (ఒక కొత్త repo create చేసి)
2. Jenkins job configuration లో, **Definition**: "Pipeline script from
   SCM" select చేయి
3. SCM: Git, Repository URL: నీ GitHub repo లింక్ పెట్టు
4. Save → Build Now

ఇది real-world లో అందరూ వాడే విధానం — నీ code, Jenkinsfile తో సహా, Git
repo లోనే ఉంటుంది.

## Step 4: Build Now click చేయి

Job page లో ఎడమవైపు **"Build Now"** click చేయి. కింద Build History లో
కొత్త build number (#1) కనిపిస్తుంది.

## Step 5: Console Output చూడు

Build number మీద క్లిక్ చేసి, **"Console Output"** select చేయి. ఇలాంటిది
కనిపించాలి:

```
[Pipeline] stage (Build)
Installing dependencies...
+ pip install -r requirements.txt
...
[Pipeline] stage (Test)
Running tests...
+ pytest -v
test_app.py::test_add PASSED
test_app.py::test_subtract PASSED
test_app.py::test_multiply PASSED
test_app.py::test_divide PASSED
test_app.py::test_divide_by_zero PASSED
[Pipeline] stage (Package)
Build and tests passed. Ready to package/deploy!
Pipeline succeeded! ✅
```

అన్ని 5 tests **PASSED** గా కనిపిస్తే — నీ మొదటి working Jenkins
pipeline complete! 🎉

## Practice Ideas (తర్వాత try చేయడానికి)

1. `app.py` లో ఒక కొత్త function add చేయి (e.g., `power(a, b)`), దానికి
   `test_app.py` లో ఒక test రాయి, మళ్ళీ Build Now click చేసి pass
   అవుతుందో చూడు
2. `test_app.py` లో ఏదైనా test ని **intentionally fail** అయ్యేలా మార్చి
   (e.g., `assert add(2, 3) == 999`), pipeline **red (failed)** గా
   మారుతుందో చూడు — ఇది CI/CD యొక్క అసలైన value: bugs ని తొందరగా
   పట్టుకోవడం
3. `Jenkinsfile` లో ఒక కొత్త stage add చేయి, e.g., `stage('Lint')` —
   code style check చేసేది
