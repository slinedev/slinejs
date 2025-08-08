# Sline.js: JavaScript-Based Template Engine

## 1\. 概述

### 1.1. 项目目标

本项目旨在为 SHOPLINE 的 Sline 模板引擎创建一个功能对等、高性能且符合人体工程学的 JavaScript 实现（以下简称 Sline.js）。该引擎的核心目标是让前端开发者能够在 JavaScript 环境中（浏览器和 Node.js）使用 Sline 语法，**以赋能客户端动态交互和支持无头 (Headless) 电商架构**。

### 1.2. 核心原则

  * **性能优先:** 最小化初始加载体积，最大化运行时渲染速度。预编译 (AOT) 为首选生产策略。
  * **完全兼容:** 严格遵循 SHOPLINE 官方文档定义的 Sline 语法、标签、过滤器和对象。
  * **开发者体验 (DX):** 提供简洁的 API、清晰的错误信息和通过 Source Maps 实现的无缝调试。
  * **安全第一:** 不使用 `eval` 或 `new Function()`，确保引擎在设计上是 CSP 友好的。

### 1.3. 项目定位与边界 (vs. `shopline-cli`)

为避免功能重叠，Sline.js 的定位必须被严格界定：

  * **Sline.js 是一个模板渲染库，而非主题开发工具。** 它的核心职责是解析和渲染 Sline 模板字符串。
  * **`shopline-cli` 负责主题的完整生命周期管理。** 这包括主题初始化、本地开发服务器（提供完整的服务器端上下文）、资源管理、配置 (`.json`) 文件处理以及部署。
  * **Sline.js 不处理主题结构。** 它不理解 `sections`, `layout`, `config` 等目录结构，也不处理 `{% schema %}` 标签。
  * **Sline.js 的应用场景始于 `shopline-cli` 的边界之外。** 它专注于 `shopline-cli` 无法覆盖的领域：**浏览器端的动态渲染**。

### 1.4. 范围

  * **`@sline/core`:** 包含解析器、AST 解释器和运行时的核心 NPM 包。
  * **`@sline/compiler`:** (可选，作为性能优化) 基于 `@sline/core` 构建的 Node.js 命令行工具 (CLI)，用于将模板预编译成 JS 函数。
  * **`@sline/vite-plugin`:** 一个 Vite 插件，用于在现代前端项目中无缝集成 Sline 模板的预编译。
  * **`sline.js` (UMD):** 一个用于浏览器 `<script>` 标签直接引用的 UMD 构建版本。

-----

## 2\. 系统架构与数据流程设计

Sline.js 采用双模执行架构，优先考虑安全性和灵活性。

### 2.1. 架构组件概览

  * **解析器 (Parser):** 接收 Sline 模板字符串，输出一个结构化的**抽象语法树 (AST)**。
  * **AST 解释器 (Interpreter):** 在运行时遍历 AST，结合数据上下文，动态生成 HTML 输出。**这是 JIT 模式的核心，取代了不安全的 `new Function()`。**
  * **代码生成器 (Code Generator):** (用于 AOT 模式) 遍历 AST，将其编译成一个优化后的、可执行的 JavaScript **渲染函数 (Render Function)** 的字符串表示。
  * **运行时 (Runtime):** 提供所有 Sline 内置过滤器和引擎执行时所需的辅助函数。

### 2.2. 数据流程 1：运行时编译 (JIT - 浏览器)

此模式提供了最大的灵活性，允许在客户端动态编译模板，且无需放宽 CSP。

**流程图:**
`Sline 模板字符串` -\> `engine.parseAndRender(str, data)` -\> **[1. 解析器]** -\> `AST` -\> \*\*\*\* -\> `HTML 字符串 (Promise)`

**步骤分解:**

1.  **实例化引擎:** `const engine = new Sline(options);`
2.  **渲染:** 开发者调用 `engine.parseAndRender(templateString, data)`。
3.  **内部处理:**
      * **解析器**将字符串转换为 AST（带缓存）。
      * **AST 解释器**异步遍历 AST，结合数据和运行时辅助函数，生成最终的 HTML 字符串。
4.  **返回结果:** 方法返回一个解析为 HTML 字符串的 Promise。

### 2.3. 数据流程 2：预编译 (AOT - 构建时)

这是**生产环境的推荐方案**，将所有编译开销完全前置。

**流程图:**
`源文件 (*.html)` -\> **[Vite 插件]** -\> `sline-compiler` -\> **[1. 解析器]** -\> `AST` -\> **[2. 代码生成器]** -\> `JS 模块字符串` -\> **[Vite 处理]** -\> `前端 import` -\> `渲染函数` -\> `fn(data)` -\> `HTML 字符串`

**步骤分解:**

1.  **文件处理:** Vite 在构建时遇到一个配置为由 Sline.js 处理的 `.html` 文件。
2.  **插件调用:** `@sline/vite-plugin` 插件的 `transform` 钩子被触发。
3.  **编译:** 插件内部调用 `@sline/compiler`，后者执行解析和代码生成步骤，最终输出一个标准的 ES 模块字符串。
4.  **模块化:** Vite 接收这个 JS 模块字符串并将其集成到最终的构建产物中。
5.  **使用:** 前端代码可以直接 `import` 这个编译好的渲染函数，实现零运行时编译开销。

### 2.4. 核心架构决策：

`Sline.js` 的一个关键特性是它不使用 `eval` 或 `new Function()`，从而在设计上杜绝了代码注入风险。Sline.js 将采纳这一**安全优先**的原则：

  * **JIT 模式:** `Sline.compile()` 将不再使用 `new Function()`。取而代之的是，它会返回一个函数，该函数在执行时由一个安全的**运行时解释器**来遍历预先生成的 AST。这虽然会带来微小的性能开销，但完全消除了对 `'unsafe-eval'` CSP 策略的依赖，使引擎在默认情况下更加安全。
  * **AOT 模式:** 预编译模式不受影响，因为它在构建时生成的是静态、安全的 JavaScript 代码，本身就是 CSP 友好的。

-----

## 3\. 核心组件技术实现与依赖

### 3.1. 解析器 (Parser)

  * **职责:** 将 Sline 模板字符串转换为 AST，并提供精确的错误位置信息。
  * **技术选型:** **Peggy.js**
  * **依赖:** `peggy`
  * **理由:** Peggy.js 的声明式 PEG 语法非常适合定义 Sline 的嵌套块级结构和表达式，生成的解析器性能良好且错误报告清晰。
  * **关键实现细节:**
      * **语法兼容:** 解析器必须严格遵循官方文档定义的语法，包括：
          * **注释:** `{{!-- comment --}}` [2]。
          * **字符串:** 支持 `""` (标准), `''` (单字符), 和 `` ` `` (原生多行) 三种定义方式 [2, 5]。
          * **操作符优先级:** 必须正确处理 `&&` 高于 `||` 的优先级 [5]。
          * **空白控制:** 必须能解析并处理 `{~` 和 `~}` 标记 [5]。
  * **AST 节点接口 (示例):**
    ```typescript
    interface Node { type: string; loc: { start: number; end: number; }; }
    interface TextNode extends Node { type: 'Text'; content: string; }
    interface ExpressionNode extends Node {
      type: 'Expression';
      path: string; // e.g., ['product', 'variants', 0, 'title']
      filters: { name: string; args: any };
      escaped: boolean; // true for {{ }}, false for {{{ }}}
    }
    interface ForBlockNode extends Node {
      type: 'ForBlock';
      iterable: ExpressionNode;
      alias: string;
      body: Node;
    }
    ```

### 3.2. 代码生成器 (Code Generator)

  * **职责:** 遍历 AST，生成一个作为 ES 模块的 JavaScript 渲染函数字符串。
  * **技术选型:** **纯 TypeScript 实现**
  * **依赖:** `source-map-js` (用于生成 Source Maps)
  * **实现细节:**
      * 创建一个 `CodeGenerator` 类，通过访问者模式遍历 AST。
      * **作用域处理:** 生成的代码**不得**实现 `../` 父级作用域访问，因为 Sline 明确禁止此功能 [2]。
      * **条件逻辑:** `if` 标签的编译必须遵循 Sline 的真值/假值规则（例如，空数组和空字符串均为 `true`）[5]。
      * **`for` 循环:** 编译 `for` 循环时，必须生成对 `runtime.helpers.pushForloop()` 和 `runtime.helpers.popForloop()` 的调用，以正确管理 `forloop` 和 `forloop.parentloop` 对象 [6]。
      * **Source Maps:** 在生成代码的每一步，都记录下与原始 `.html` 文件中对应位置的映射关系，以提供无缝的调试体验。

### 3.3. 运行时 (Runtime)

  * **职责:** 提供所有内置过滤器和渲染函数所需的辅助函数。
  * **技术选型:** **纯 TypeScript 实现**
  * **依赖 (截至 2025 年最新技术栈):**
      * **`date-fns`:** 用于实现功能强大且国际化友好的 `date` 过滤器。它以其模块化、tree-shaking 支持和不变性而成为现代首选 。
      * **`just-safe-get`:** 一个轻量级库，用于实现安全的深层属性访问 `lookup` 辅助函数。
  * **API 结构:**
    ```typescript
    export const runtime = {
      filters: {
        money: (val) => { /*... */ },
        upcase: (val) => String(val).toUpperCase(),
        //... 实现 Sline 官方文档列出的所有过滤器 [1]
      },
      helpers: {
        lookup: (context, path) => { /*... */ },
        escape: (str) => { /*... */ },
        applyFilter: (name, value,...args) => { /*... */ },
        isTruthy: (val) => { /* 实现 Sline 特定的真值判断逻辑 */ },
        forloopStack:, // 用于管理 forloop 上下文
        pushForloop: (iterable) => { /*... */ },
        popForloop: () => { /*... */ },
      }
    };
    ```

### 3.4. Extensibility API

  * **职责:** 允许开发者注册自定义过滤器，增强引擎功能。
  * **API 设计:**
      * `Sline.registerFilter(name: string, filterFn: Function)`: 向全局运行时注册一个新的过滤器。
      * `Sline.compile(str, { filters: {... } })`: 在单次编译时传入局部过滤器。

-----

## 4\. 关键问题解决方案

### 4.1. 在 `.html` 文件中处理 Sline 语法

  * **AOT 方案 (Vite 插件 - `@sline/vite-plugin`):**

      * **机制:** 创建一个 Vite 插件，利用 `transform` 钩子拦截特定文件。
      * **触发方式:** 开发者通过在 `import` 语句后添加查询参数来显式触发转换。这是 Vite 生态中的最佳实践，避免了对所有 `.html` 文件进行不必要的处理。
        ```javascript
        import renderFunction from './my-template.html?sline';
        ```
      * **插件逻辑 (`vite.config.js`):**
        ```javascript
        // vite.config.js
        import sline from '@sline/vite-plugin';
        export default {
          plugins: [sline()]
        }

        // @sline/vite-plugin/index.js (示意)
        export default function slinePlugin() {
          return {
            name: 'vite-plugin-sline',
            transform(code, id) {
              if (!id.endsWith('.html?sline')) return null;
              const compiledJs = compiler.compile(code, { module: true, sourceMap: true });
              return { code: compiledJs.code, map: compiledJs.map };
            }
          };
        }
        ```

  * **JIT 方案 (浏览器):**

      * **机制:** 采用 `<script type="text/sline-template">` 标签包裹模板。
      * **优点:** 与 Handlebars 等传统引擎用法一致，开发者易于理解。完全隔离了模板内容与主 HTML DOM。

### 4.2. 浏览器端 API 设计

`sline.js` UMD 包将在全局暴露一个 `Sline` 对象。

  * **`Sline.compile(templateString, options?)`:**

      * 核心 JIT 编译函数。
      * 内部维护一个基于模板字符串的缓存，避免重复编译。
      * 返回一个渲染函数。

  * **`Sline.render(templateId, data, targetSelector)`:**

      * 便捷的“三合一”函数：获取模板、编译（带缓存）、渲染并安全地更新 DOM。

  * **`Sline.registerFilter(name, fn)`:**

      * 全局注册自定义过滤器。

-----

## 5\. 测试策略

### 5.1. 测试框架与环境

  * **框架:** **Vitest v3.x**
  * **环境:** **happy-dom** (用于需要 DOM 的测试，比 jsdom 更快)
  * **理由 (截至 2025 年):** Vitest 已成为与 Vite 集成的首选测试框架，以其出色的性能、与 ES 模块的原生兼容性以及与 Vite 配置的无缝共享而著称，相比 Jest 在现代前端工作流中通常提供更优的开发者体验 。
  * **配置:** 在 `vitest.config.ts` 中配置测试环境和全局设置。Vitest 将自动读取 `vite.config.ts`，确保测试环境与构建环境一致。

### 5.2. 测试分类与实施

1.  **单元测试 (Unit Tests):**

      * **目标:** `runtime` 中的每一个过滤器和辅助函数。
      * **工具:** Vitest 的 `describe`, `it`, `expect`。
      * **示例 (`runtime.test.ts`):**
        ```typescript
        import { expect, it, describe } from 'vitest';
        import { runtime } from '@sline/core';

        describe('upcase filter', () => {
          it('should convert a string to uppercase', () => {
            expect(runtime.filters.upcase('hello')).toBe('HELLO');
          });
        });
        ```

2.  **集成测试 (Integration Tests):**

      * **目标:** 验证解析器和代码生成器的协作。
      * **方法:** 输入 Sline 字符串，断言生成的 JS 代码字符串符合预期结构。
      * **工具:** Vitest 的 `toMatchInlineSnapshot`，可以清晰地展示生成的代码。

3.  **端到端快照测试 (E2E Snapshot Tests):**

      * **目标:** 验证从模板到最终 HTML 输出的完整流程，覆盖所有语法特性。
      * **流程:**
        1.  创建 `test/fixtures` 目录，存放成对的 `.html` 模板和 `.json` 数据文件。
        2.  测试脚本遍历 fixtures，对每一对进行编译和渲染。
        3.  使用 Vitest 的 `toMatchSnapshot()` 将 HTML 输出与快照文件进行对比 。
      * **示例 (`e2e.test.ts`):**
        ```typescript
        import { expect, it, describe } from 'vitest';
        import Sline from '@sline/core';
        import fs from 'fs';
        import path from 'path';

        describe('E2E Template Rendering', () => {
          const template = fs.readFileSync(path.join(__dirname, 'fixtures/product.html'), 'utf-8');
          const data = JSON.parse(fs.readFileSync(path.join(__dirname, 'fixtures/product.json'), 'utf-8'));

          it('should render the product template correctly', () => {
            const templateFn = Sline.compile(template);
            const output = templateFn(data);
            expect(output).toMatchSnapshot();
          });
        });
        ```

-----

## 6\. 安全考量

  * **`unsafe-eval` 与 CSP:**

      * **核心决策:** Sline.js 的 JIT 模式将**不使用 `new Function()` 或 `eval()`** [4]。
      * **结果:** Sline.js 在默认情况下是 CSP 友好的，无需 `'unsafe-eval'`，从而显著提升了其安全性。

  * **跨站脚本 (XSS):**

      * **默认转义:** `{{ expression }}` 必须对所有非 `SafeString` 对象进行严格的 HTML 转义 [1, 3]。
      * **原始输出:** `{{{ expression }}}` 是一个明确的“危险”操作。文档必须指导用户，**绝不能**将未经服务器端严格净化和验证的用户输入通过此方式输出 [1, 3]。

-----

## 7\. 战略意义与开发者体验提升

对于 SAAS 电商平台而言，店铺主题的开发体验直接关系到其生态系统的活力和商家的最终满意度。实现一个纯前端渲染的 Sline.js 引擎，其意义远不止是技术栈的补充，更是一项战略性的赋能。

### 7.1. 项目的战略意义 (The "Why")

1.  **解锁现代化的用户体验 (UX):** 当前，店铺主题主要依赖服务器端渲染，每次交互（如筛选商品、更新购物车）都需要重新加载页面。Sline.js 将使主题能够构建**客户端驱动的动态功能**，例如：

      * **AJAX 商品筛选:** 无刷新更新商品列表。
      * **动态商品选项:** 实时更新价格、库存和图片，而无需重新加载页面。
      * **交互式购物车/迷你购物车:** 即时增删商品和更新总价。
      * **“快速查看”弹窗:** 在不离开列表页的情况下查看商品详情。
        这些功能是现代电商网站的标配，Sline.js 为实现它们提供了官方、标准化的途径。

2.  **性能优化与架构现代化:** Sline.js 允许主题开发者采用“数据驱动”的更新模式。他们可以通过 Storefront API 获取轻量级的 JSON 数据，然后在客户端使用 Sline.js 高效地渲染 HTML。这相比于通过 AJAX 获取预渲染的整个 HTML 片段，**显著减少了网络负载**，加快了动态交互的响应速度。

3.  **统一技术栈与降低认知负荷:** Sline.js 意味着开发者只需学习**一种模板语言**，即可同时覆盖服务器端和客户端的渲染需求。这种同构能力消除了在 Sline 和另一种前端技术（如 React/Vue 或手写 JS）之间切换的上下文成本。

4.  **增强平台生态竞争力:** 提供一套与 Shopify Liquid 对标的、功能强大的前端开发工具，将使 SHOPLINE 平台对高水平的主题开发者和机构更具吸引力，从而催生出功能更丰富、体验更流畅的优质主题。

### 7.2. 对开发者体验的帮助 (The "How")

1.  **代码复用:** 开发者可以编写一个 `product-card.html` 的 Sline 模板片段（Snippet/Component），它既可以在服务器端用于初始页面加载，也可以在客户端用于 AJAX 刷新或“快速查看”弹窗的渲染。**一次编写，处处运行**，极大地提高了开发效率和可维护性。

2.  **告别繁琐的 DOM 操作:** 在没有 Sline.js 的情况下，开发者需要手动拼接 HTML 字符串或使用 `document.createElement` 等原生 DOM API 来构建动态内容。这个过程不仅繁琐、易错，而且难以维护。Sline.js 以一种声明式、结构化的方式完美地解决了这个问题。

      * **之前 (手动拼接):**
        ```javascript
        const html = '<div class="product">' + '<h3>' + product.name + '</h3>' + '</div>';
        container.innerHTML += html;
        ```
      * **之后 (Sline.js):**
        ```javascript
        const html = productCardTemplate({ product: product });
        container.insertAdjacentHTML('beforeend', html);
        ```

3.  **清晰的关注点分离:** Sline.js 强化了 HTML 结构 (在 `.html` 模板中) 与 JavaScript 逻辑 (获取数据、调用渲染) 之间的分离。这使得代码更易于阅读、调试和团队协作。设计师可以专注于模板的 HTML 和 CSS，而开发者可以专注于数据流和交互逻辑。

4.  **官方支持与稳定性:** 为客户端渲染提供一个官方支持的、与平台核心模板语言完全兼容的引擎，给了开发者信心。他们不必再依赖于可能与 Sline 语法有细微差异的第三方库，或担心自己的实现会因平台更新而失效。
