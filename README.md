🍳 Transform Recipes

Battle-tested AWS Transform Custom recipes for automating cloud best practices. From observability to security, transform your infrastructure in minutes with atx.

What is This?

Transform Recipes is a community-driven collection of transformation definitions for AWS Transform Custom. Each recipe automates the implementation of cloud best practices—turning hours or days of manual configuration into minutes of AI-powered automation.

Whether you need to instrument your application for observability, harden security configurations, or implement governance policies, there's likely a recipe here that can help.

Why Use Transform Recipes?

Save Time: What used to take days of reading documentation, writing code, and debugging now takes minutes.

Battle-Tested: Every recipe has been validated in real-world scenarios and includes prerequisites, validation steps, and expected outcomes.

Best Practices Built-In: Each transformation implements AWS Well-Architected Framework principles and industry best practices.

Learn by Example: See exactly what changes AWS Transform Custom makes to your infrastructure and application code.

Quick Start
Install AWS Transform Custom CLI
Browse Available Recipes

Recipes are organized by the six pillars of cloud operations:
• Observability - CloudWatch Application Signals, OpenTelemetry, X-Ray, logging
• Security - Secrets management, IAM policies, WAF, encryption
• Governance - AWS Config rules, tagging standards, compliance
• Resiliency - Multi-AZ deployments, backups, disaster recovery
• Operations - Automated patching, incident response, log aggregation
• Financial Management - Cost allocation, budget alerts, resource optimization
Choose a Recipe

Each recipe includes:
• transformationdefinition.md - The transformation definition for atx
• README.md - Overview, use case, and quick start guide
• prerequisites.md - Required setup and permissions
• validation.md - How to verify the transformation succeeded
Run the Transformation
Validate the Results

Follow the validation steps in validation.md to confirm everything is working correctly.

Featured Recipes

CloudWatch Application Signals - ECS Daemon Strategy
Automatically instrument your ECS applications with OpenTelemetry and configure CloudWatch Application Signals for end-to-end observability.

Time: ~20 minutes | Cost: ~$0.70 | View Recipe →

Contributing

We welcome contributions! Whether you have a new recipe to share or improvements to existing ones, please see our Contributing Guidelines.

Quick contribution checklist:
• [ ] Recipe follows the standard structure
• [ ] Includes all required files (transformation_definition.md, README.md, prerequisites.md, validation.md)
• [ ] Tested with atx CLI
• [ ] Includes cost and time estimates
• [ ] Documents validation steps

Recipe Structure

Each recipe follows a consistent structure to make them easy to use and contribute:

See docs/recipe-structure.md for detailed guidelines.

Documentation
• Getting Started - First time using Transform Recipes?
• Recipe Structure - How recipes are organized
• Best Practices - Tips for writing great transformations
• Troubleshooting - Common issues and solutions

Cost Transparency

Each recipe includes estimated costs for running the transformation. Actual costs may vary based on:
• Your AWS region
• Application complexity
• Existing infrastructure
• AWS Transform Custom pricing in your region

Always test transformations in a non-production environment first.

Community & Support
• Questions? Open a GitHub Discussion
• Found a bug? Submit an Issue
• Want to contribute? Read our Contributing Guidelines
• Need help? Check the Troubleshooting Guide

Related Resources
• AWS Transform Custom
• AWS Well-Architected Framework
• AWS Builder Center
• CloudWatch Application Signals
• AWS Distro for OpenTelemetry

License

This repository is licensed under the MIT License. See LICENSE for details.

Transformation definitions and code samples are provided under the MIT-0 License (no attribution required).

Acknowledgments

Special thanks to all contributors who have shared their transformation recipes and helped make cloud best practices accessible to everyone.

Built with ❤️ by the AWS community

Transform your infrastructure. Share your recipes. Build better together.
