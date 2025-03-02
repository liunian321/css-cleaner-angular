const fs = require('fs-extra');
const path = require('path');
const postcss = require('postcss');
const selectorParser = require('postcss-selector-parser');
const { XMLParser } = require('fast-xml-parser');

// 添加缓存机制
const fileCache = new Map();

async function processDirectory(dirPath) {
  try {
    const entries = await fs.readdir(dirPath, { withFileTypes: true });

    // 使用并行处理子目录
    await Promise.all(
      entries.map(async (entry) => {
        if (entry.isDirectory()) {
          await processDirectory(path.join(dirPath, entry.name));
        }
      }),
    );
    const validExtensions = new Set(['.html', '.css', '.less', '.scss']);

    // 处理当前目录的文件
    const files = entries.filter((e) => e.isFile());
    const fileGroups = new Map();

    // 修改文件分组逻辑
    for (const file of files) {
      // 文件名
      const name = file.name;
      // 文件扩展名
      const ext = path.extname(name).toLowerCase();
      // 文件扩展名不匹配
      if (!validExtensions.has(ext)) continue;
      // 文件名
      const base = path.basename(name, ext);
      // 文件路径
      const key = path.join(dirPath, base);

      if (!fileGroups.has(key)) {
        fileGroups.set(key, { html: null, styles: [] });
      }

      const group = fileGroups.get(key);
      if (ext === '.html') {
        group.html = path.join(dirPath, file.name);
      } else {
        group.styles.push(path.join(dirPath, file.name));
      }
    }

    // 过滤掉没有对应HTML文件的组
    const validGroups = [...fileGroups.entries()].filter(([_, group]) => {
      if (group.html) {
        return true;
      } else {
        // 如果有CSS文件但没有HTML文件，输出提示信息
        if (group.styles.length > 0) {
          console.log(
            `Skipping ${group.styles.join(', ')}: No corresponding HTML file found`,
          );
        }
        return false;
      }
    });

    // 并行处理文件组
    await Promise.all(
      validGroups.map(async ([key, group]) => {
        if (group.html && group.styles.length > 0) {
          try {
            await processFileGroup(group);
          } catch (error) {
            console.error(`Error processing group ${key}:`, error);
          }
        }
      }),
    );
  } catch (error) {
    console.error(`Error processing directory ${dirPath}:`, error);
  }
}

async function processFileGroup({ html, styles }) {
  // 使用缓存避免重复读取文件
  let htmlContent;
  if (fileCache.has(html)) {
    htmlContent = fileCache.get(html);
  } else {
    htmlContent = await fs.readFile(html, 'utf8');
    fileCache.set(html, htmlContent);
  }

  const parser = new XMLParser({
    ignoreAttributes: false,
    attributeNamePrefix: '@_',
    allowBooleanAttributes: true,
    parseAttributeValue: true,
  });
  const jsonObj = parser.parse(htmlContent);
  const usedClasses = new Set();

  // 递归遍历解析后的对象
  function traverse(node) {
    if (node && typeof node === 'object') {
      // 处理 class 属性
      if (node['@_class']) {
        node['@_class'].split(/\s+/).forEach((cls) => {
          if (cls) {
            usedClasses.add(cls);
          }
        });
      }

      // 处理 ngClass 属性
      const ngClassAttr =
        node['@_ngClass'] ||
        node['@_[ngClass]'] ||
        node['@_data-ng-class'] ||
        node['@_[data-ng-class]'] ||
        node['@_[ng-class]'] ||
        node['@_[data-ngClass]'];
      if (ngClassAttr) {
        const matches = ngClassAttr.match(/'([^']+)'|"([^"]+)"/g);
        if (matches) {
          matches.forEach((match) => {
            const className = match.replace(/['"]/g, '');
            if (
              className &&
              !['true', 'false', 'null', 'undefined'].includes(className)
            ) {
              usedClasses.add(className);
              console.log(`Found class in ngClass: ${className}`);
            }
          });
        }
      }

      // 递归处理子节点
      for (const key in node) {
        if (Array.isArray(node[key])) {
          node[key].forEach(traverse);
        } else if (typeof node[key] === 'object') {
          traverse(node[key]);
        }
      }
    }
  }

  traverse(jsonObj);

  // 并行处理样式文件
  await Promise.all(
    styles.map(async (stylePath) => {
      let cssContent;
      if (fileCache.has(stylePath)) {
        cssContent = fileCache.get(stylePath);
      } else {
        cssContent = await fs.readFile(stylePath, 'utf8');
        fileCache.set(stylePath, cssContent);
      }

      // 如果CSS内容为空，跳过处理
      if (!cssContent.trim()) {
        return;
      }

      // 清理CSS
      const cleaned = await cleanCSS(cssContent, usedClasses);

      // 只有在内容发生变化时才创建备份并写入新内容
      if (cleaned !== cssContent) {
        // 先创建备份文件
        const backupPath = path.format({
          dir: path.dirname(stylePath),
          name: path.basename(stylePath, path.extname(stylePath)) + '.backup',
          ext: path.extname(stylePath),
        });

        // 将原始内容写入备份文件
        await fs.writeFile(backupPath, cssContent);
        console.log(`Generated backup: ${backupPath}`);

        // 将清理后的内容写入原文件
        await fs.writeFile(stylePath, cleaned);
        console.log(`Updated file: ${stylePath}`);
      } else {
        console.log(`No changes needed for: ${stylePath}`);
      }
    }),
  );
}

// 优化CSS清理插件
function createCleanPlugin(usedClasses, ignoredPrefixes = []) {
  return {
    postcssPlugin: 'clean-unused',
    Rule(rule) {
      // 只处理以.开头的选择器
      if (!rule.selector.startsWith('.')) {
        return;
      }

      // 使用postcss-selector-parser来更准确地解析选择器
      let shouldKeep = false;
      selectorParser((selectors) => {
        selectors.walkClasses((cls) => {
          if (
            usedClasses.has(cls.value) ||
            ignoredPrefixes.some((prefix) => cls.value.startsWith(prefix))
          ) {
            shouldKeep = true;
          } else {
            for (const cls of usedClasses) {
              // console.log(cls, cls.value, cls.includes(cls.value));

              if (cls.includes(cls.value)) {
                shouldKeep = true;
                break;
              }
            }
          }
        });
      }).processSync(rule.selector);

      if (!shouldKeep) {
        // console.log(`Removing rule: ${rule.selector}`);
        rule.remove();
      } else {
        // console.log(`Keeping rule: ${rule.selector}`);
      }
    },
  };
}

// 添加命令行参数处理
const targetDir = process.argv[2] || process.cwd();
if (!fs.existsSync(targetDir)) {
  console.error(`Directory ${targetDir} does not exist`);
  process.exit(1);
}

processDirectory(targetDir).catch((error) => {
  console.error('Fatal error:', error);
  process.exit(1);
});

async function cleanCSS(css, usedClasses) {
  return postcss([
    createCleanPlugin(usedClasses, ['ant', 'ng', 'nz', 'custom-prefix']),
  ])
    .process(css)
    .then((result) => result.css);
}
