# Contributing to System Design Interview Prep

Thank you for your interest in contributing! This repository aims to help engineers prepare for system design interviews using Alex Xu's methodology.

## How to Contribute

### Types of Contributions

1. **New System Design Case Studies**
   - Follow the standard template in `06-templates/system-design-template.md`
   - Include all sections: requirements, estimation, API, database, architecture, deep dive, bottlenecks
   - Add Mermaid diagrams for visual clarity
   - Ensure accuracy and interview relevance

2. **Improvements to Existing Designs**
   - Clarify explanations
   - Add or improve diagrams
   - Fix technical inaccuracies
   - Add alternative approaches or trade-offs

3. **Code Implementations**
   - Python implementations of algorithms and building blocks
   - Clean, well-commented code
   - Include docstrings and type hints
   - Add example usage

4. **Documentation**
   - Fix typos and grammar
   - Improve clarity
   - Add references and links
   - Update README files

5. **Interview Tips and Resources**
   - Share interview experiences (anonymized)
   - Add company-specific insights
   - Curate useful resources

## Contribution Guidelines

### Content Standards

1. **Accuracy** - Ensure technical correctness
2. **Clarity** - Write for understanding, not to impress
3. **Completeness** - Cover all aspects of the design
4. **Practicality** - Focus on interview-relevant content
5. **Attribution** - Credit sources and references

### Writing Style

- Use clear, concise language
- Explain trade-offs and alternatives
- Include concrete numbers in estimations
- Use bullet points and tables for readability
- Follow the Alex Xu methodology structure

### Diagrams

- Use Mermaid for architecture diagrams (renders on GitHub)
- Keep diagrams simple and focused
- Label all components clearly
- Use consistent notation
- Include both high-level and detailed diagrams

### Code Style

- Follow PEP 8 for Python code
- Add type hints (Python 3.8+)
- Include docstrings for functions and classes
- Keep functions focused and modular
- Add comments for complex logic

## Submission Process

### Before Submitting

1. Check if a similar contribution already exists
2. Review the template for the content type
3. Test any code you're submitting
4. Proofread for typos and clarity
5. Ensure diagrams render correctly

### Pull Request Process

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/new-design`)
3. Make your changes following the guidelines
4. Commit with clear messages (`git commit -m "Add: Netflix system design"`)
5. Push to your fork (`git push origin feature/new-design`)
6. Open a Pull Request with a clear description

### PR Description Template

```markdown
## Description
Brief description of what you're adding or changing

## Type of Change
- [ ] New system design case study
- [ ] Improvement to existing design
- [ ] Code implementation
- [ ] Documentation update
- [ ] Bug fix

## Checklist
- [ ] Follows the standard template
- [ ] Includes diagrams (if applicable)
- [ ] Code is tested (if applicable)
- [ ] No typos or grammatical errors
- [ ] References are cited

## Additional Context
Any additional information or context
```

## System Design Template Structure

All system designs should include:

1. **Problem Statement & Requirements**
   - Functional requirements
   - Non-functional requirements
   - Out of scope items

2. **Back-of-the-Envelope Estimation**
   - Traffic estimates (QPS, DAU)
   - Storage estimates
   - Bandwidth estimates

3. **API Design**
   - REST/GraphQL endpoints
   - Request/response formats

4. **Data Model & Database Schema**
   - Tables/collections
   - Indexes
   - Database technology choice

5. **High-Level Design**
   - Architecture diagram
   - Component overview
   - Data flow

6. **Detailed Component Design**
   - Deep dive into 2-3 key components
   - Technology choices
   - Algorithms and data structures

7. **Identifying and Resolving Bottlenecks**
   - Single points of failure
   - Performance bottlenecks
   - Solutions (replication, sharding, caching)

8. **Trade-offs and Alternatives**
   - Design decisions explained
   - Alternative approaches considered

9. **Monitoring, Metrics & Alerts** (optional but recommended)
   - Key metrics
   - Logging strategy
   - Alert conditions

10. **Follow-up Questions & Extensions**
    - How would you scale to 10x?
    - How would you handle failures?
    - How would you add feature X?

## Code of Conduct

### Our Standards

- Be respectful and constructive
- Focus on what's best for learning
- Accept constructive criticism gracefully
- Help others learn and grow

### Unacceptable Behavior

- Plagiarism or uncredited copying
- Deliberately incorrect or misleading information
- Harassment or discriminatory language
- Spam or promotional content

## Questions or Suggestions?

- Open an issue for questions
- Start a discussion for ideas
- Tag maintainers for urgent matters

## Recognition

Contributors will be acknowledged in the project. Significant contributions may be highlighted in the README.

## License

By contributing, you agree that your contributions will be used for educational purposes and may be freely distributed with attribution.

---

Thank you for helping make this resource better for everyone! Your contributions help engineers worldwide prepare for their dream jobs.
