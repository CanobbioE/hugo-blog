---
title: "Resume"
date: 2023-03-17T15:37:50+04:00
draft: false
resources:
    - /Canobbio%20Edoardo%20-%20CV.pdf
---
[Download the latest PDF version of my resume](/Canobbio%20Edoardo%20-%20CV.pdf).


# Work Experience

{{< resume/section 
    title=`Senior Software Engineer (Platform)`
    company=`IndyKite`
    location=`Remote`
    from=`AUGUST 2023`
    to=`PRESENT`
>}}
<p>Single-handedly migrated the company Go codebase to FIPS 140-3 compliance, ensuring cryptographic standards and boosting customer confidence.</p>
<p>Led the adoption of Google Cloud Secret Manager, securely migrating sensitive data and significantly enhancing the company's security posture, contributing to successful SOC 2 compliance.</p>
<p>Enhanced service observability by improving distributed tracing, resulting in an 80% reduction in mean time to recovery.</p>
<p>Executed a seamless migration from Neo4j to CockroachDB, using feature flags and parameterized tests to maintain uptime. This enabled multi-region availability and improved platform resilience.</p>
<p>Refactored high-volume Temporal workflows, replacing 200+ GB JSON-based batch operations with concurrent data streaming, achieving a 10x reduction in memory usage and processing time.</p>
{{< /resume/section >}}

{{< resume/section 
    title=`Senior Software Engineer (Payments)`
    company=`Glofox`
    location=`Remote`
    from=`SEPTEMBER 2021`
    to=`AUGUST 2023`
>}}
<p>Designed and delivered new features within a serverless architecture, applying Agile practices such as user story mapping, requirement slicing, and “three amigos” planning, accelerating delivery and improving cross-functional alignment among developers, designers, and QA.</p>
<p>Mentored teammates across experience levels, supporting onboarding and technical growth, which contributed to team expansion and improved developer retention.</p>
<p>Led the refactoring of critical microservices to enhance reliability and maintainability, resolving 150+ high-priority, customer-facing issues in six months and significantly improving system stability and customer satisfaction.</p>
<p>Promoted testing excellence through rigorous code reviews and test-driven development, leading to a 20% increase in test coverage and overall improved code quality.</p>
<p>Documented gRPC APIs and integrated SwaggerUI via a REST reverse proxy, improving developer experience and boosting internal tooling efficiency through easier API discovery and usage.</p>
{{< /resume/section >}}


{{< resume/section 
    title=`Product Engineer (Platform)`
    company=`Glofox`
    location=`Remote`
    from=`SEPTEMBER 2020`
    to=`SEPTEMBER 2021`
>}}
<p>Re-architected an underperforming messaging service by splitting it into multiple scalable runtimes and leveraging AWS SNS and SQS, reducing queue wait times from over 90 minutes to ~12 minutes during peak traffic, and significantly improving throughput and responsiveness.</p>
<p>Integrated end-to-end tests into the release pipeline via CircleCI for concurrent Golang microservices, increasing release confidence and accelerating delivery velocity across the team.</p>
<p>Implemented an auto-scaling mechanism for concurrent Go SQS workers, reducing the company’s AWS monthly costs by $272. </p>
{{< /resume/section >}}


{{< resume/section 
    title=`Full Stack Software Developer`
    company=`Ti&m`
    location=`Frankfurt, Germany`
    from=`OCTOBER 2019`
    to=`AUGUST 2020`
>}}
<p>Re-architected a slow image recognition ML service into a performant front-end application using TensorFlow.js and Angular 2+, reducing average recognition time from over 1000 ms to ~150 ms, significantly enhancing the user experience.</p>
<p>Automated and accelerated deployments across multiple Firebase projects by integrating GitLab CI/CD pipelines, enabling faster, more consistent, and reliable release cycles.</p>
<p>Optimized ETL data workflows and improved system reliability for a major German bank, enhancing backend data processing and playing a key role in securing the client through strengthened product capabilities.</p>
<p>Implemented a scalable WebView mobile app using React Native with a Firebase serverless backend, delivering an efficient, full-stack mobile solution aligned with client requirements.</p>
{{< /resume/section >}}

{{< resume/section 
    title=`Security Intern`
    company=`Certimeter`
    location=`Turin, Italy`
    from=`OCTOBER 2017`
    to=`NOVEMBER 2017`
>}}
<p>Secured web APIs with Java EE and IBM Security Access Manager, strengthening the security framework to comply with enterprise standards and organizational policies.</p>
<p>Led the migration of Assembly Line systems for a leading Italian enterprise to modern infrastructure leveraging Tivoli Directory Integrator and Microsoft Servers, driving enhanced system integration.</p>
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