## Making Sense of Code Scanning with CodeQL

It's no secret that GitHub's code scanning tool, CodeQL, can help organizations of any size develop software faster and more securely, all without needing to leave GitHub.com, and there's no shortage of awesome content explaining what this looks like from the developer or organization owner's perspectives using the tool. However, something that I've struggled with is figuring out a simple yet thorough explanation of the mechanisms behind _how_ CodeQL delivers its code scanning alerts (something even deeper than statements like "it queries a relational database representation of the repo's code, which gets generated with every trigger event"). So that's why I'm writing this, and if it's something you'd like to learn, Keep Reading.

One final note: my focus here is on _how_ CodeQL works, not why it's valuable. To better understand the value propisition of using code scanning to help SHIFT LEFT, then you can read more about it [here](https://github.blog/2020-08-27-secure-at-every-step-putting-devsecops-into-practice-with-code-scanning/).

### The Basic Steps of CodeQL
1. Preparing the code by creating a CodeQL database
2. Running CodeQL queries against the database
3. Interpreting/presenting the query results

For my deep-dive here, I'm going to tackle each one of these in turn, starting with the first. Remember that in the context of code scanning, the running of this entire process is defined in the workflow YAML file inside your .github/workflows directory in your repo. The setup to get started with this has been automated within GitHub and occurs when you first turn on code scanning under the "Security" tab of your repo. See here:


(Explain how the workflow file basically works, and reference that the whole code scanning process can also be done in a repo from the CLI if you have the right packages included)

### 1. Preparing the Code by Creating a CodeQL Database

The most basic explanation you'll find regarding this step is that to create a database, CodeQL first extracts a single relational representation of each source file in the codebase. But what does that entail?

#### The Extraction Process

The conversion from source code to a querable database is done by a special program called an extractor, and to perform this process on any given language, an extractor built specifically for that language is needed. As I'll explain later, the ambiguities present in certain programming languages (such as Ruby) make it harder to write extractors, as one of the key functions of an extractor is to _parse_ the code, which is more difficult for more ambiguous languages.

Beyond which language is being used, there are two different scenarios for the extraction process, both covered below.

##### Extraction for Compiled Languages

Extraction in this case works by monitoring the normal build process. Each time a compiler is invoked to process a source file, a copy of that file is made, and all relevant information about the source code is collected. This includes syntactic data about the abstract syntax tree and semantic data about name binding and type information.

##### Extraction for Non-Compiled Languages:

For non-compiled languages, the extractor runs directly on the source code, resolving dependencies to give an accurate representation of the codebase.

In both scenarios, a parse tree is created to represent the source code. What exactly does that mean though?

It all starts with 

(use a super basic example representing some source code, show how it's turned into a parse tree and then a db)




















You can use the [editor on GitHub](https://github.com/nomdal/understandingcodeql/edit/main/README.md) to maintain and preview the content for your website in Markdown files.

Whenever you commit to this repository, GitHub Pages will run [Jekyll](https://jekyllrb.com/) to rebuild the pages in your site, from the content in your Markdown files.

### Markdown

Markdown is a lightweight and easy-to-use syntax for styling your writing. It includes conventions for

```markdown
Syntax highlighted code block

# Header 1
## Header 2
### Header 3

- Bulleted
- List

1. Numbered
2. List

**Bold** and _Italic_ and `Code` text

[Link](url) and ![Image](src)
```

For more details see [Basic writing and formatting syntax](https://docs.github.com/en/github/writing-on-github/getting-started-with-writing-and-formatting-on-github/basic-writing-and-formatting-syntax).

### Jekyll Themes

Your Pages site will use the layout and styles from the Jekyll theme you have selected in your [repository settings](https://github.com/nomdal/understandingcodeql/settings/pages). The name of this theme is saved in the Jekyll `_config.yml` configuration file.

### Support or Contact

Having trouble with Pages? Check out our [documentation](https://docs.github.com/categories/github-pages-basics/) or [contact support](https://support.github.com/contact) and we’ll help you sort it out.
