Se Jenkins chiede password:


```console
docker exec -it jenkins-local-jenkins-1 cat /var/jenkins_home/secrets/initialAdminPassword
```

Il nome del container qui è "jenkins-local-jenkins-1 cat" ma potrebbe essere diverso, ricontrollare

## Lista plugin da installare su Jenkins

- Tutti i plugin base consigliati
- Docker pipeline

## Generazione PAT token GitHub

Affinchè la connessione tra Jenkins e GitHub sia possibile è necessario accedere sul proprio profilo GitHub e generare un PAT token fine-grained.
[Qui](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens) la guida alla generazione. I permessi da includere sono:
