<div align="center">
  <h1>Chirpy Jekyll Theme</h1>
  <p>A minimal and powerful Jekyll theme for GitHub Pages.</p>
  <a href="https://jekyllrb.com/docs/installation/">Jekyll Documentation</a> •
  <a href="https://github.com/cotes2020/chirpy-starter">Chirpy Starter</a>
</div>

---

## 🚀 Quick Start

Follow these steps to set up your site using the Chirpy Jekyll Theme.

### Prerequisites

Ensure the following tools are installed on your system:

- [Ruby](https://www.ruby-lang.org/en/)
- [RubyGems](https://rubygems.org/pages/download)
- [Jekyll](https://jekyllrb.com/docs/installation/)
- [Bundler](https://bundler.io/)
- [Git](https://git-scm.com/)

### Step 1: Create a New Repository

1. Use the [**Chirpy Starter**](https://github.com/cotes2020/chirpy-starter/generate) to generate a new repository.
2. Name the repository `<GH_USERNAME>.github.io`, where `<GH_USERNAME>` is your GitHub username.

### Step 2: Install Dependencies

Navigate to the root directory of your site and install the required dependencies:

```bash
bundle install
```

### Step 3: Run the Local Server

To preview your site locally, run the following command:

with draft post,
```bash
bundle exec jekyll serve --draft
```

without draft post,
```bash
bundle exec jekyll serve
```

Alternatively, you can use Docker:

```bash
docker run -it --rm \
    --volume="$PWD:/srv/jekyll" \
    -p 4000:4000 jekyll/jekyll \
    jekyll serve
```

Once the server is running, open your browser and visit: [http://localhost:4000](http://localhost:4000).

---

## 📂 Project Structure

- `_posts/`: Add your blog posts here.
- `_config.yml`: Configure your site settings.
- `assets/`: Store images, CSS, and JavaScript files.
- `README.md`: Documentation for your project.

---

## 🌟 Features

- Minimal design with a focus on content.
- Fully responsive and mobile-friendly.
- Easy integration with GitHub Pages.
- Support for categories, tags, and archives.

---

## 🛠️ Customization

To customize your site, edit the `_config.yml` file. You can update:

- Site title, description, and URL.
- Social media links.
- Theme settings and plugins.

---

## 📜 License

This project is licensed under the [MIT License](LICENSE).

---

## 🤝 Contributing

Contributions are welcome! Feel free to open issues or submit pull requests to improve this project.

---

## 📧 Contact

For questions or support, please reach out via [GitHub Issues](https://github.com/<GH_USERNAME>/<REPO_NAME>/issues).