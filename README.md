Hugo ile Minimal Blog

Bu klasör Hugo ile oluşturulmuş basit bir blog iskeletidir. Aşağıdaki adımlarla projeyi yerel olarak çalıştırabilirsiniz.

Gereksinimler:
- Hugo yüklü olmalı. macOS için:

```bash
brew install hugo
```

Yerel sunucuyu başlatmak için:

```bash
cd /Users/emre/Projects/hugo
hugo server -D
```

Üretim için statik dosyaları oluşturmak:

```bash
cd /Users/emre/Projects/hugo
hugo -D
# Çıktı `public/` klasöründe oluşur; bunu barındırmaya yükleyin.
```

Oluşturduğum dosyalar:
- [config.toml](config.toml)
- [content/posts/hello-world.md](content/posts/hello-world.md)
- [layouts/_default/baseof.html](layouts/_default/baseof.html)
- [layouts/_default/single.html](layouts/_default/single.html)
- [layouts/index.html](layouts/index.html)
- [static/css/styles.css](static/css/styles.css)

İsterseniz ben Hugo kurulumunu ve `hugo server` çalıştırmayı da terminalde deneyeyim.

GitHub Pages ile otomatik deploy (GitHub):

1) Yeni bir GitHub deposu oluşturun (ör. `emre/hugo`).

2) Yerel projeyi git ile başlatın ve uzak repo'yu ekleyin, ardından `main` dalına push edin:

```bash
cd /Users/emre/Projects/hugo
git init
git add .
git commit -m "Initial Hugo scaffold"
git branch -M main
git remote add origin git@github.com:YOUR_USERNAME/YOUR_REPO.git
git push -u origin main
```

3) Workflow otomatik olarak `main`'e her push yaptığınızda çalışacak, Hugo'yu derleyip `gh-pages` dalına deploy edecektir. GitHub Pages ayarlarında (Settings → Pages) kaynağı `gh-pages` branch olarak seçin.

Notlar:
- Workflow, `GITHUB_TOKEN` kullanır; ekstra secret gerekmez.
- Özel domain kullanacaksanız bir `CNAME` dosyası ekleyin ve Pages ayarlarından domain'i ekleyin.

