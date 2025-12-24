---
layout: default
title: About
permalink: /about/
---

<div class="about-page">
    <header class="about-header">
        <img src="{{ site.author.avatar }}" alt="{{ site.author.name }}" class="avatar-large">
        <h1>{{ site.author.name }}</h1>
        <p class="role">Software Architect at Microsoft</p>
    </header>

    <div class="about-content">
        <p>I spend my days designing distributed systems that (hopefully) don't fall over at scale. Currently focused on Azure Logic Apps and AI platform at Microsoft, where I've been solving cloud infrastructure puzzles for over a decade.</p>
        
        <p>Before Logic Apps, I worked on Azure Resource Manager — the layer that handles every deployment you've ever clicked "Create" on in Azure. These days, I'm more interested in how AI is reshaping cloud infrastructure.</p>
        
        <p>When I'm not debugging production or arguing about system design, I tinker with open source projects — mostly around Azure extensibility and local-first AI assistants.</p>
        
        <p>I studied at UT Dallas and NIT Trichy back when "cloud" still mostly meant weather.</p>
        
        <p>Based in Seattle. Opinions are my own.</p>

        <h2>Projects</h2>
        <div class="project-grid">
            <a href="https://github.com/laveeshb/logicapps-mcp" target="_blank" rel="noopener noreferrer" class="project-card">
                <h3>Logic Apps MCP Server</h3>
                <p>MCP server for Azure Logic Apps</p>
                <span class="tech-tag">TypeScript</span>
            </a>
            <a href="https://github.com/laveeshb/azure-functions-sqs-extension" target="_blank" rel="noopener noreferrer" class="project-card">
                <h3>Azure Functions SQS Extension</h3>
                <p>Multi-language Azure Functions bindings for Amazon SQS</p>
                <span class="tech-tag">C#</span>
            </a>
            <a href="https://github.com/laveeshb/krakenly" target="_blank" rel="noopener noreferrer" class="project-card">
                <h3>Krakenly</h3>
                <p>A fully local, privacy-focused AI assistant that runs entirely on your machine</p>
                <span class="tech-tag">Python</span>
            </a>
        </div>

        <h2>Education</h2>
        <ul class="education-list">
            <li>
                <strong>The University of Texas at Dallas</strong><br>
                Master of Science, Computer Science (2008 – 2010)
            </li>
            <li>
                <strong>National Institute of Technology, Tiruchirappalli</strong><br>
                Bachelor of Technology (2002 – 2006)
            </li>
        </ul>
    </div>
</div>
