# Tools Ecosystem for Frontend Microservices

Comprehensive guide to the tools and technologies that power modern frontend microservices architecture.

## 🛠️ Core Build Tools

### Webpack 5 with Module Federation

```javascript
// webpack.config.js for Shell Application
const ModuleFederationPlugin = require('@module-federation/webpack');

module.exports = {
  mode: 'development',
  devServer: {
    port: 3000,
    hot: true,
    historyApiFallback: true,
  },
  
  plugins: [
    new ModuleFederationPlugin({
      name: 'shell',
      filename: 'remoteEntry.js',
      remotes: {
        user_service: 'user@http://localhost:3001/remoteEntry.js',
        product_service: 'product@http://localhost:3002/remoteEntry.js',
        cart_service: 'cart@http://localhost:3003/remoteEntry.js'
      },
      shared: {
        react: { singleton: true, eager: true },
        'react-dom': { singleton: true, eager: true },
        '@shared/components': { singleton: true }
      }
    })
  ]
};

// webpack.config.js for Microservice
module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'user_service',
      filename: 'remoteEntry.js',
      exposes: {
        './UserApp': './src/App',
        './UserProfile': './src/components/UserProfile',
        './userStore': './src/store'
      },
      shared: {
        react: { singleton: true },
        'react-dom': { singleton: true }
      }
    })
  ]
};
```

### Vite with Federation

```javascript
// vite.config.js with @originjs/vite-plugin-federation
import { defineConfig } from 'vite';
import { federation } from '@originjs/vite-plugin-federation';

export default defineConfig({
  plugins: [
    federation({
      name: 'user_service',
      filename: 'remoteEntry.js',
      exposes: {
        './UserApp': './src/App.vue'
      },
      shared: {
        vue: { singleton: true },
        '@shared/utils': {}
      }
    })
  ],
  
  build: {
    target: 'esnext',
    minify: false,
    cssCodeSplit: false
  }
});
```

## 📦 Monorepo Management

### NX Configuration

```json
// nx.json
{
  "version": 2,
  "projects": {
    "shell": "apps/shell",
    "user-service": "apps/user-service",
    "product-service": "apps/product-service",
    "shared-components": "libs/shared/components",
    "shared-utils": "libs/shared/utils"
  },
  
  "targetDefaults": {
    "build": {
      "dependsOn": ["^build"],
      "cache": true
    },
    "test": {
      "cache": true
    }
  },

  "generators": {
    "@nrwl/react": {
      "application": {
        "style": "styled-components",
        "linter": "eslint",
        "bundler": "webpack"
      }
    }
  }
}
```

### Custom NX Executors

```javascript
// tools/executors/module-federation/executor.js
const { execSync } = require('child_process');

module.exports = async function runExecutor(options, context) {
  const { projectName } = context;
  const config = context.workspace.projects[projectName];
  
  // Generate federation config
  const federationConfig = generateFederationConfig(options, config);
  
  // Start dev server with proper ports
  const port = await findAvailablePort(options.port);
  
  console.log(`Starting ${projectName} on port ${port}`);
  
  try {
    execSync(`webpack serve --config ${federationConfig} --port ${port}`, {
      stdio: 'inherit',
      cwd: config.root
    });
    
    return { success: true };
  } catch (error) {
    return { success: false, error: error.message };
  }
};

function generateFederationConfig(options, config) {
  // Dynamic configuration generation based on project
  return `webpack.federation.${config.projectType}.js`;
}
```

## 🎨 Development Tools

### Module Federation Dashboard

```javascript
// tools/dashboard/federationDashboard.js
const express = require('express');
const { WebSocketServer } = require('ws');

class FederationDashboard {
  constructor(port = 4000) {
    this.app = express();
    this.port = port;
    this.services = new Map();
    this.setupRoutes();
  }

  start() {
    const server = this.app.listen(this.port, () => {
      console.log(`🎛️  Federation Dashboard running on port ${this.port}`);
    });

    this.wss = new WebSocketServer({ server });
    this.setupWebSocket();
  }

  setupRoutes() {
    this.app.get('/api/services', (req, res) => {
      res.json(Array.from(this.services.entries()).map(([name, info]) => ({
        name,
        ...info
      })));
    });

    this.app.post('/api/services/:name/health', (req, res) => {
      const { name } = req.params;
      this.updateServiceHealth(name, req.body);
      res.json({ success: true });
    });

    this.app.use(express.static('tools/dashboard/public'));
  }

  setupWebSocket() {
    this.wss.on('connection', (ws) => {
      ws.send(JSON.stringify({
        type: 'services',
        data: Array.from(this.services.entries())
      }));
    });
  }

  updateServiceHealth(name, health) {
    this.services.set(name, {
      ...this.services.get(name),
      health,
      lastSeen: new Date()
    });

    this.broadcast({
      type: 'health_update',
      service: name,
      health
    });
  }

  broadcast(message) {
    this.wss.clients.forEach(client => {
      client.send(JSON.stringify(message));
    });
  }
}

module.exports = FederationDashboard;
```

### Hot Module Replacement for Federation

```javascript
// tools/hmr/federationHMR.js
class FederationHMR {
  constructor() {
    this.moduleMap = new Map();
    this.setupHMR();
  }

  setupHMR() {
    if (module.hot) {
      module.hot.accept(['./src/components'], () => {
        this.reloadFederatedModules();
      });
    }

    // Watch for remote module changes
    this.watchRemoteModules();
  }

  watchRemoteModules() {
    const EventSource = window.EventSource || require('eventsource');
    const sse = new EventSource('/federation-hmr');

    sse.onmessage = (event) => {
      const data = JSON.parse(event.data);
      if (data.type === 'module_update') {
        this.handleRemoteUpdate(data);
      }
    };
  }

  async handleRemoteUpdate(data) {
    const { moduleName, remoteUrl } = data;
    
    // Clear module cache
    delete window[moduleName];
    
    // Reload remote entry
    await this.loadRemoteEntry(remoteUrl);
    
    // Notify React about the update
    this.triggerReactUpdate(moduleName);
  }

  async loadRemoteEntry(url) {
    const script = document.createElement('script');
    script.src = `${url}?t=${Date.now()}`;
    
    return new Promise((resolve, reject) => {
      script.onload = resolve;
      script.onerror = reject;
      document.head.appendChild(script);
    });
  }

  triggerReactUpdate(moduleName) {
    window.dispatchEvent(new CustomEvent('federation:module_update', {
      detail: { moduleName }
    }));
  }
}

export const federationHMR = new FederationHMR();
```

## 🔧 Build and Deployment Tools

### Custom Build Scripts

```javascript
// scripts/build-all.js
const { spawn } = require('child_process');
const fs = require('fs');
const path = require('path');

class MicroserviceBuildManager {
  constructor() {
    this.services = this.discoverServices();
    this.buildOrder = this.calculateBuildOrder();
  }

  discoverServices() {
    const appsDir = path.join(__dirname, '../apps');
    return fs.readdirSync(appsDir)
      .filter(name => fs.statSync(path.join(appsDir, name)).isDirectory())
      .map(name => ({
        name,
        path: path.join(appsDir, name),
        dependencies: this.getDependencies(name)
      }));
  }

  getDependencies(serviceName) {
    const packagePath = path.join(__dirname, '../apps', serviceName, 'package.json');
    if (!fs.existsSync(packagePath)) return [];
    
    const pkg = JSON.parse(fs.readFileSync(packagePath, 'utf8'));
    const deps = {
      ...pkg.dependencies,
      ...pkg.devDependencies
    };

    return Object.keys(deps).filter(dep => dep.startsWith('@shared/'));
  }

  calculateBuildOrder() {
    // Topological sort for build dependencies
    const visited = new Set();
    const result = [];

    const visit = (service) => {
      if (visited.has(service.name)) return;
      
      service.dependencies.forEach(dep => {
        const depService = this.services.find(s => s.name === dep.replace('@shared/', ''));
        if (depService) visit(depService);
      });

      visited.add(service.name);
      result.push(service);
    };

    this.services.forEach(visit);
    return result;
  }

  async buildAll() {
    console.log('🏗️  Building services in optimal order...');
    
    for (const service of this.buildOrder) {
      await this.buildService(service);
    }

    console.log('✅ All services built successfully');
  }

  async buildService(service) {
    console.log(`Building ${service.name}...`);
    
    return new Promise((resolve, reject) => {
      const build = spawn('npm', ['run', 'build'], {
        cwd: service.path,
        stdio: 'pipe'
      });

      build.stdout.on('data', (data) => {
        console.log(`[${service.name}] ${data}`);
      });

      build.stderr.on('data', (data) => {
        console.error(`[${service.name}] ${data}`);
      });

      build.on('close', (code) => {
        if (code === 0) {
          console.log(`✅ ${service.name} built successfully`);
          resolve();
        } else {
          reject(new Error(`Build failed for ${service.name}`));
        }
      });
    });
  }
}

// Usage
const buildManager = new MicroserviceBuildManager();
buildManager.buildAll().catch(console.error);
```

### Docker Multi-Stage Build

```dockerfile
# Dockerfile.microservice
FROM node:18-alpine as builder

WORKDIR /app

# Copy package files
COPY package*.json ./
COPY apps/user-service/package*.json ./apps/user-service/
COPY libs/shared/ ./libs/shared/

# Install dependencies
RUN npm ci --only=production

# Copy source code
COPY apps/user-service ./apps/user-service
COPY libs/shared ./libs/shared

# Build the service
RUN npm run build:user-service

# Production stage
FROM nginx:alpine

# Copy built files
COPY --from=builder /app/dist/apps/user-service /usr/share/nginx/html

# Copy nginx configuration
COPY nginx/microservice.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

## 🧪 Testing Tools Integration

### Custom Test Runner

```javascript
// tools/test-runner/microserviceTestRunner.js
const Jest = require('jest');
const { glob } = require('glob');

class MicroserviceTestRunner {
  constructor() {
    this.services = this.discoverTestableServices();
  }

  discoverTestableServices() {
    return glob.sync('apps/*/src/**/*.test.{js,ts,tsx}')
      .reduce((acc, testFile) => {
        const serviceName = testFile.split('/')[1];
        if (!acc[serviceName]) {
          acc[serviceName] = [];
        }
        acc[serviceName].push(testFile);
        return acc;
      }, {});
  }

  async runTests(options = {}) {
    const { service, watch, coverage } = options;

    if (service) {
      return this.runServiceTests(service, { watch, coverage });
    }

    return this.runAllTests({ watch, coverage });
  }

  async runServiceTests(serviceName, options) {
    const testFiles = this.services[serviceName];
    if (!testFiles) {
      throw new Error(`No tests found for service: ${serviceName}`);
    }

    const jestConfig = {
      testMatch: testFiles,
      collectCoverage: options.coverage,
      watchAll: options.watch,
      setupFilesAfterEnv: ['<rootDir>/jest.setup.js']
    };

    const result = await Jest.runCLI(jestConfig, [process.cwd()]);
    return result;
  }

  async runAllTests(options) {
    console.log('🧪 Running all microservice tests...');
    
    const results = {};
    
    for (const [serviceName, testFiles] of Object.entries(this.services)) {
      console.log(`\n📋 Testing ${serviceName}...`);
      
      try {
        results[serviceName] = await this.runServiceTests(serviceName, options);
      } catch (error) {
        results[serviceName] = { success: false, error };
      }
    }

    this.generateTestReport(results);
    return results;
  }

  generateTestReport(results) {
    const summary = Object.entries(results).map(([service, result]) => ({
      service,
      passed: result.results?.numPassedTests || 0,
      failed: result.results?.numFailedTests || 0,
      coverage: result.results?.coverageMap?.getCoverageSummary()?.toSummary() || {}
    }));

    console.table(summary);
    
    // Generate detailed HTML report
    this.generateHTMLReport(summary);
  }

  generateHTMLReport(summary) {
    const html = `
      <!DOCTYPE html>
      <html>
        <head>
          <title>Microservices Test Report</title>
          <style>
            body { font-family: Arial, sans-serif; }
            .service { margin: 20px 0; padding: 15px; border: 1px solid #ddd; }
            .passed { color: green; }
            .failed { color: red; }
          </style>
        </head>
        <body>
          <h1>Microservices Test Report</h1>
          ${summary.map(s => `
            <div class="service">
              <h2>${s.service}</h2>
              <p class="passed">Passed: ${s.passed}</p>
              <p class="failed">Failed: ${s.failed}</p>
            </div>
          `).join('')}
        </body>
      </html>
    `;

    require('fs').writeFileSync('test-report.html', html);
  }
}

module.exports = MicroserviceTestRunner;
```

## 📊 Monitoring and Analytics Tools

### Performance Monitoring

```javascript
// tools/monitoring/performanceMonitor.js
class MicroservicePerformanceMonitor {
  constructor() {
    this.metrics = new Map();
    this.setupPerformanceObserver();
    this.setupCustomMetrics();
  }

  setupPerformanceObserver() {
    if (typeof PerformanceObserver === 'undefined') return;

    const observer = new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        this.recordMetric(entry.name, {
          duration: entry.duration,
          startTime: entry.startTime,
          entryType: entry.entryType
        });
      }
    });

    observer.observe({
      entryTypes: ['navigation', 'resource', 'measure']
    });
  }

  setupCustomMetrics() {
    // Module Federation loading metrics
    this.monitorModuleFederation();
    
    // Runtime performance metrics
    this.monitorRuntimePerformance();
  }

  monitorModuleFederation() {
    const originalDefine = window.__webpack_require__?.define;
    if (!originalDefine) return;

    window.__webpack_require__.define = (...args) => {
      const start = performance.now();
      const result = originalDefine.apply(this, args);
      const duration = performance.now() - start;

      this.recordMetric('module_federation_load', {
        duration,
        module: args[0]
      });

      return result;
    };
  }

  monitorRuntimePerformance() {
    // Memory usage tracking
    setInterval(() => {
      if (performance.memory) {
        this.recordMetric('memory_usage', {
          used: performance.memory.usedJSHeapSize,
          total: performance.memory.totalJSHeapSize,
          limit: performance.memory.jsHeapSizeLimit
        });
      }
    }, 10000);

    // Bundle size tracking
    this.trackBundleSizes();
  }

  trackBundleSizes() {
    const observer = new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        if (entry.name.includes('remoteEntry.js')) {
          this.recordMetric('bundle_size', {
            name: entry.name,
            transferSize: entry.transferSize,
            encodedBodySize: entry.encodedBodySize
          });
        }
      }
    });

    observer.observe({ entryTypes: ['resource'] });
  }

  recordMetric(name, data) {
    if (!this.metrics.has(name)) {
      this.metrics.set(name, []);
    }

    this.metrics.get(name).push({
      ...data,
      timestamp: Date.now()
    });

    // Send to analytics service
    this.sendToAnalytics(name, data);
  }

  sendToAnalytics(metricName, data) {
    // Batch metrics to reduce network calls
    if (!this.analyticsQueue) {
      this.analyticsQueue = [];
      setTimeout(() => this.flushAnalytics(), 5000);
    }

    this.analyticsQueue.push({
      metric: metricName,
      data,
      timestamp: Date.now()
    });
  }

  async flushAnalytics() {
    if (!this.analyticsQueue?.length) return;

    try {
      await fetch('/api/analytics/metrics', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          metrics: this.analyticsQueue
        })
      });
    } catch (error) {
      console.warn('Failed to send analytics:', error);
    }

    this.analyticsQueue = [];
  }

  getMetrics(name) {
    return this.metrics.get(name) || [];
  }

  getMetricsSummary() {
    const summary = {};
    
    for (const [name, data] of this.metrics.entries()) {
      const durations = data.map(d => d.duration).filter(Boolean);
      
      summary[name] = {
        count: data.length,
        avgDuration: durations.reduce((a, b) => a + b, 0) / durations.length,
        maxDuration: Math.max(...durations),
        minDuration: Math.min(...durations)
      };
    }

    return summary;
  }
}

export const performanceMonitor = new MicroservicePerformanceMonitor();
```

## 🔍 Debugging Tools

### Federation Debugger

```javascript
// tools/debugger/federationDebugger.js
class FederationDebugger {
  constructor() {
    this.isEnabled = process.env.NODE_ENV === 'development';
    this.setup();
  }

  setup() {
    if (!this.isEnabled) return;

    this.interceptModuleLoading();
    this.createDebugPanel();
    this.setupConsoleCommands();
  }

  interceptModuleLoading() {
    const originalImport = window.__webpack_require__;
    if (!originalImport) return;

    window.__webpack_require__ = new Proxy(originalImport, {
      apply: (target, thisArg, args) => {
        const moduleId = args[0];
        console.log(`🔍 Loading module: ${moduleId}`);
        
        try {
          const result = target.apply(thisArg, args);
          console.log(`✅ Module loaded: ${moduleId}`);
          return result;
        } catch (error) {
          console.error(`❌ Module failed: ${moduleId}`, error);
          throw error;
        }
      }
    });
  }

  createDebugPanel() {
    const panel = document.createElement('div');
    panel.id = 'federation-debug-panel';
    panel.style.cssText = `
      position: fixed;
      top: 10px;
      right: 10px;
      width: 300px;
      background: #1a1a1a;
      color: white;
      padding: 15px;
      border-radius: 5px;
      font-family: monospace;
      font-size: 12px;
      z-index: 10000;
      display: none;
    `;

    panel.innerHTML = `
      <h3>🔍 Federation Debug</h3>
      <div id="loaded-modules"></div>
      <div id="failed-modules"></div>
      <button onclick="federationDebugger.exportLogs()">Export Logs</button>
    `;

    document.body.appendChild(panel);

    // Toggle panel with keyboard shortcut
    document.addEventListener('keydown', (e) => {
      if (e.ctrlKey && e.shiftKey && e.key === 'D') {
        panel.style.display = panel.style.display === 'none' ? 'block' : 'none';
      }
    });
  }

  setupConsoleCommands() {
    window.federationDebugger = {
      listModules: () => {
        console.table(Object.keys(window.__webpack_require__.cache || {}));
      },
      
      reloadModule: async (moduleId) => {
        delete window.__webpack_require__.cache[moduleId];
        return window.__webpack_require__(moduleId);
      },
      
      exportLogs: () => {
        const logs = this.getLogs();
        const blob = new Blob([JSON.stringify(logs, null, 2)], {
          type: 'application/json'
        });
        
        const url = URL.createObjectURL(blob);
        const a = document.createElement('a');
        a.href = url;
        a.download = `federation-debug-${Date.now()}.json`;
        a.click();
        URL.revokeObjectURL(url);
      }
    };
  }

  getLogs() {
    return {
      timestamp: new Date().toISOString(),
      modules: Object.keys(window.__webpack_require__?.cache || {}),
      performance: performanceMonitor.getMetricsSummary(),
      errors: this.getErrorLogs()
    };
  }

  getErrorLogs() {
    // Collect error information
    return window.federationErrors || [];
  }
}

export const federationDebugger = new FederationDebugger();
```

## 📋 Tool Selection Matrix

| Category | Primary Tool | Alternative | Use Case |
|----------|-------------|-------------|----------|
| **Module Federation** | Webpack 5 | Vite Federation | Dynamic imports, runtime integration |
| **Build Tool** | Webpack | Vite, Rollup | Production builds, dev server |
| **Monorepo** | NX | Lerna, Rush | Multi-package management |
| **Package Manager** | npm | yarn, pnpm | Dependency management |
| **Bundler** | Webpack | Parcel, ESBuild | Asset bundling |
| **Dev Server** | Webpack Dev Server | Vite Dev Server | Hot reloading |
| **Testing** | Jest + Cypress | Vitest + Playwright | Unit and E2E testing |
| **Linting** | ESLint + Prettier | Biome | Code quality |
| **Type Checking** | TypeScript | Flow | Static typing |
| **CI/CD** | GitHub Actions | GitLab CI, Jenkins | Automated deployment |

## 🚀 Quick Setup Scripts

```bash
#!/bin/bash
# setup-microservices.sh

# Install global dependencies
npm install -g nx webpack-cli

# Create workspace
npx create-nx-workspace@latest microservices --preset=empty

cd microservices

# Add React support
nx add @nrwl/react

# Generate shell application
nx generate @nrwl/react:app shell --routing=true

# Generate microservices
nx generate @nrwl/react:app user-service
nx generate @nrwl/react:app product-service

# Generate shared libraries
nx generate @nrwl/react:lib shared-components
nx generate @nrwl/workspace:lib shared-utils

# Setup Module Federation configs
cp scripts/webpack.shell.js apps/shell/webpack.config.js
cp scripts/webpack.micro.js apps/user-service/webpack.config.js

echo "✅ Microservices workspace created successfully!"
echo "Run 'npm run dev:all' to start all services"
```

This comprehensive tools ecosystem provides everything needed to develop, test, build, and deploy frontend microservices efficiently. The combination of build tools, development utilities, and monitoring solutions creates a robust development environment.

- [Next Advanced patterns](11-advanced-patterns.md)