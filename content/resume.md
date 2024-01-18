---
title: "Resume"
date: 2023-03-17T15:37:50+04:00
draft: false
resources:
    - /Canobbio%20Edoardo%20-%20CV.pdf
---
[Download the latest PDF version of my CV](/Canobbio%20Edoardo%20-%20CV.pdf).


# Work Experience

{{< resume/section 
    title=`Senior Software Engineer (Platform)`
    company=`IndyKite`
    location=`Remote`
    from=`AUGUST 2023`
    to=`PRESENT`
>}}
<p> Developed new gRPC endpoints and improved existing ones. </p>
<p> Introduced the use of Google Cloud Platform's secret manager and securely migrated sensible data </p>
<p> Improved tracing capabilities across services. </p>
<p> Migrated core services from Neo4j to CockroachDB. Introduced parametrized integration tests to simplify the testing of the database migration.</p>
{{< /resume/section >}}

{{< resume/section 
    title=`Senior Software Engineer (Payments)`
    company=`Glofox`
    location=`Remote`
    from=`SEPTEMBER 2021`
    to=`AUGUST 2023`
>}}
<p> Architected and implemented new features, following Agile’s principles: slicing
requirements, planning “three amigos” sessions and leading user story mappings.</p>
<p>Onboarded and mentored junior, mid and senior level teammates.</p>
<p> Led the refactoring of multiple Serverless microservices, resolving over 150 high
priority customer impacting issues, in less than six months. </p>
<p>Established good testing practices by influencing others through code reviews and
leading by example.</p>
<p>Documented the API of a gRPC service, linking SwaggerUI to a REST reverse proxy.</p>
{{< /resume/section >}}


{{< resume/section 
    title=`Product Engineer (Platform)`
    company=`Glofox`
    location=`Remote`
    from=`SEPTEMBER 2020`
    to=`SEPTEMBER 2021`
>}}
<p> Re-architected and implemented a messaging service, separating it into multiple,
easily scalable, runtimes, using SNS and SQS. </p>
<p> As a result, the queue waiting times dropped: from over 90 minutes to around twelve,
under heavy traffic. </p>
<p> Introduced E2E tests as a CircleCI pipeline step for concurrent Golang microservices,
increasing confidence and velocity of future releases. </p>
<p> Implemented an auto-scaling mechanism for concurrent Go SQS workers, reducing the
company’s AWS monthly costs by $272. </p>
{{< /resume/section >}}


{{< resume/section 
    title=`Full Stack Software Developer`
    company=`Ti&m`
    location=`Frankfurt, Germany`
    from=`OCTOBER 2019`
    to=`AUGUST 2020`
>}}
<p>Re-designed a machine learning service into a front-end-only application, using
TensorFlowJs and Angular 2+.
Consequently, reducing the average image recognition time from over 1000 ms to
about 150 ms.</p>
<p>Hastened the Firebase deployment of multiple projects leveraging GitLab’s CI/CD.</p>
<p>Enhanced the ETL software of one of Germany’s major bank institutions, which heavily
contributed to acquiring them as a customer.</p>
<p>Developed a WebView mobile app, using Firebase as a serverless back end and React
Native.</p>
{{< /resume/section >}}

{{< resume/section 
    title=`Security Intern`
    company=`Certimeter`
    location=`Turin, Italy`
    from=`OCTOBER 2017`
    to=`NOVEMBER 2017`
>}}
<p>Developed and secured web APIs using Java EE and IBM Security Access Manager.
Facilitated Assembly Lines’ migration for a large Italian enterprise, using Tivoli
Directory Integrator and Microsoft Servers.</p>
{{< /resume/section >}}

# Education
{{< resume/static-section 
    title=`AWS Cloud Practitioner certificate`
    company=`Amazon Web Services`
    location=`Remote`
    from=`JUNE 2020`
>}}

{{< resume/section 
    title=`Developer / SCRUM Master`
    company=`Switzerland Academy`
    location=`Milan, Italy`
    from=`JANUARY 2019`
    to=`MAY 2019`
>}}
<p>As the team’s Scrum Master, learned to apply Agile principles to the development of a
React application.</p>
<p>Integrated payments Stripe’s API (cards and direct debit) and MetaMask (crypto
currencies).</p>
<p>Developed and implemented a proof of concept for a Self-Sovereign Identity service,
using Ethereum blockchain and Solidity.</p>
<p>Designed an algorithm to calculate an investment portfolio’s risk level.</p>
{{< /resume/section >}}

{{< resume/static-section 
    title=`Bachelor's Degree in Computer Science`
    company=`University of Turin`
    location=`Turin, Italy`
    from=`OCTOBER 2015`
    to=`DECEMBER 2018`
>}}


# Projects
{{< resume/section 
    title=`Team Lead`
    company=`reelo.apnetwork.it`
    location=`Remote`
>}}
<p>Developed an ELO system for the Italian Mathematics Games in Golang.
Closely worked with a team of mathematicians to design the “Reelo” algorithm.
Acted as the mentor figure for a developer intern.</p>
{{< /resume/section >}}