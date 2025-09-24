# Math Worksheet Vue Frontend

- **Interactive math rounding exercises with beautiful glassmorphism design**

Vue.js frontend connecting to a Node.js backend API for math worksheets and high score tracking.

## What This Is

**Frontend**: Vue.js app with responsive design and smooth animations  
**Backend**: Connects to [math-worksheet-backend](https://github.com/nishanajihah/math-worksheet-backend) API  
**Architecture**: Decoupled - frontend and backend work independently

## Key Features

- **Beautiful UI**: Glassmorphism design with earth-tone colors  
- **Fully Responsive**: Works on desktop, tablet, and mobile  
- **Interactive Math**: Rounding questions with multiple choice answers  
- **Leaderboard**: Real-time high scores from all users  
- **Fast**: Built with Vite for quick development and builds

## Tech Stack

- **Vue 3** + Composition API
- **Vite** for fast builds
- **Axios** for API calls
- **Modular CSS** architecture
- **Vercel** ready deployment

## Quick Start

**Prerequisites**: Node.js 18+, [Backend API](https://github.com/nishanajihah/math-worksheet-backend) running

```bash
# Clone and install
git clone https://github.com/nishanajihah/math-worksheet-vue.git
cd math-worksheet-vue
npm install

# Development
npm run dev

# Production build
npm run build
```

## Project Structure

```text
src/
├── css/           # Modular CSS (variables, layout, responsive, etc.)
├── App.vue        # Main component  
└── main.js        # Entry point
```

## How It Works

1. **Connects to backend API** at `localhost:3000` for questions and scores
2. **Loads math questions** and displays them in responsive grid
3. **Tracks user progress** and validates answers in real-time  
4. **Submits scores** to leaderboard when complete

## Responsive Design

- **Desktop**: 3-column grid with sidebar
- **Tablet**: 2-column layout  
- **Mobile**: Single column, touch-friendly

## Deployment

**Vercel** (Recommended): Import from GitHub → Auto-deploys  
**Manual**: `npm run build` → Upload `dist/` folder

## Contributing

1. Fork the repo
2. Create feature branch  
3. Run `npm run lint`
4. Submit pull request
5. **Commit changes**: `git commit -am 'Add some feature'`
6. **Push to branch**: `git push origin feature/new-feature`
7. **Create Pull Request**

### **Code Style Guidelines**

- Use Vue 3 Composition API
- Follow ESLint configuration
- Maintain modular CSS architecture
- Ensure mobile-first responsive design
- Add comments for complex logic

## Troubleshooting

### **Common Issues**

**API Connection Failed:**

- Ensure backend server is running on port 3000
- Check CORS configuration in backend
- Verify API endpoint URLs

**Build Errors:**

- Clear node_modules: `rm -rf node_modules && npm install`
- Update dependencies: `npm update`
- Check Node.js version compatibility

**Responsive Layout Issues:**

## Show Your Support

If you found this project helpful, please give it a ⭐ on GitHub!

[![GitHub stars](https://img.shields.io/github/stars/nishanajihah/math-worksheet-vue?style=social)](https://github.com/nishanajihah/math-worksheet-vue/stargazers)

## License & Contact

MIT License | Develop by [Nisha Najihah](https://github.com/nishanajihah)  
Email: <nishanajihah.dev@gmail.com>

---
