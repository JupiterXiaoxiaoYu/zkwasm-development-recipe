# Welcome to zkWasm Development Recipe

The zkWasm Development Recipe focuses on building trustless blockchain applications using zkWasm and related technologies. It will start from scratch, from understanding blockchain and zero-knowledge proofs, to learning the basics and architecture of zkWasm, and then to developing a complete application, allowing you to transform from a beginner to an advanced zkWasm developer.

Of course, this Development Recipe can serve as both an introductory tutorial for beginners and a reference manual for developing zkWasm applications. Each chapter corresponds to a step in developing a full-stack zkWasm application.

!!! note
    This is still a work in progress. We will keep updating it with the latest information and best practices.

## Learning Experience

**Here's what you can expect from this recipe:**

- **Workflow**: A smooth development learning experience following a "learn - design - develop - test - deploy" workflow
- **Tools**: Useful tools and guidance for developing games, web, and backend applications 
- **Guidance**: Step-by-step code examples and practical guidance

**Content Format:**

- **Text**: Clear and concise explanations to help you understand the concepts and principles
- **Code**: Step-by-step code examples to guide you through the development process
- **Video**: Visual explanations to help you understand the concepts and principles
- **Quiz**: Test your understanding of the concepts and principles
- **Supplemental Resources**: Additional resources to further enhance your learning
- **FAQ**: Answers to frequently asked questions

**You will build skills and confidence in the following areas:**

- Blockchain and zkWasm engineering
- Application design
- Frontend development with React, TypeScript, and Web3 libraries
- Backend development with Rust and zkWasm 
- Environment setup, testing, debugging, and deployment

This will equip you to build your own zkWasm applications from the ground up.

**Upon completing this Development Recipe, you'll have several exciting paths to explore:**

- **Hackathons**: Put your skills to the test by participating in hackathons across various ecosystems and showcasing your zkWasm prowess
- **Entrepreneurship**: Embark on an entrepreneurial journey by building your own innovative products and starting a company, with the support of Delphinus Labs
- **Ecosystem Contribution**: Contribute to the growth of the zkWasm ecosystem by creating essential tools and libraries, with grants from Delphinus Labs
- **Career Development**: Launch your career as a sought-after full-stack developer in the thriving blockchain industry
- **Academic Research**: Delve deeper into the academic realm by pursuing research on zkWasm and pushing the boundaries of this cutting-edge technology
- **Discover Opportunities**: Discover countless other opportunities to apply your newfound expertise and make your mark in the world of blockchain and zero-knowledge proofs

## Ready to get started? 

Here are a few steps you should follow to ensure a great development experience:

### Join the Discord Community
[Delphinus Labs Discord Community](https://discord.com/invite/delphinuslab) 

- Ask questions, share your ideas, and get help from the community. 
- "The best way to learn is by teaching.” If you see others who are stuck, help them out! 
- There are also many events and activities organized by the community which you can participate in.

### Join the Telegram Dev Chat 

[Please DM The Developer Relations Team](https://t.me/jupxiao) 

- Direct Message the developer relations team to be invited into the telegram chat group.
- You can get help from the developer relations team, also you will be able to connect with other developers. 
- Please also give us your feedback and suggestions on how to improve the development experience.

**Let's dive in!**

---

## Getting Started

<div class="grid-wrapper">
    <a href="Quick%20Tutorial.html" class="grid-box">
        <h3>🚀 Quick Tutorial</h3>
        <p>Get started quickly with a hands-on tutorial</p>
        <span class="grid-link">Start Building →</span>
    </a>
    
    <a href="Core%20Concepts.html" class="grid-box">
        <h3>🔰 Core Concepts</h3>
        <p>Learn the fundamental concepts of blockchain application development</p>
        <span class="grid-link">Start Learning →</span>
    </a>
    
    <a href="Setup%20Environment.html" class="grid-box">
        <h3>⚙️ Setup Guide</h3>
        <p>Set up your development environment</p>
        <span class="grid-link">Get Ready →</span>
    </a>

    <a href="zkWasm%20Overview.html" class="grid-box">
        <h3>🚀 zkWasm Overview</h3>
        <p>Learn the basics of zkWasm</p>
        <span class="grid-link">Start Learning →</span>
    </a>
</div>

<style>
.grid-wrapper {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
    gap: 24px;
    margin: 32px 0;
}

.grid-box {
    display: block;
    padding: 24px;
    background: #ffffff;
    border: 1px solid #e1e4e8;
    border-radius: 8px;
    text-align: left;
    text-decoration: none;
    color: inherit;
    transition: all 0.3s ease;
}

.grid-box:hover {
    border-color: var(--md-primary-fg-color);
    box-shadow: 0 4px 12px rgba(0,0,0,0.1);
    text-decoration: none;
    transform: translateY(-2px);
}

.grid-box h3 {
    margin-top: 0;
    margin-bottom: 12px;
    color: var(--md-primary-fg-color);
    font-size: 1.2rem;
}

.grid-box p {
    color: #586069;
    margin-bottom: 16px;
    line-height: 1.5;
}

.grid-link {
    display: inline-flex;
    align-items: center;
    color: var(--md-primary-fg-color);
    font-weight: 500;
    font-size: 0.9rem;
}

.grid-link:after {
    content: "→";
    margin-left: 4px;
    transition: transform 0.2s ease;
}

.grid-box:hover .grid-link:after {
    transform: translateX(4px);
}

/* Dark mode support */
[data-md-color-scheme="slate"] .grid-box {
    background: #2b2b2b;
    border-color: #404040;
}

[data-md-color-scheme="slate"] .grid-box p {
    color: #9e9e9e;
}
</style>