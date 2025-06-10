<template>
    <div class="container">
        <div v-if="currentScreen === 'home'" class="flex-center flex-column screen" id="home-screen">
            <h1>Quick Quiz Game</h1>
            <button @click="startQuiz" class="btn">Play</button>
            <button @click="showHighScoresFromHome" class="btn secondary">High Scores</button>
        </div>

        <div v-if="currentScreen === 'game'" id="game-screen" class="screen flex-column">
            <div class="hud">
                <div class="hud-item">
                    <p id="progressText" class="hud-prefix">Question</p>
                    <h1 class="hud-main-text">{{ questionCounter }}/{{ MAX_QUESTIONS }}</h1>
                </div>
                <div class="hud-item">
                    <p class="hud-prefix">Score</p>
                    <h1 class="hud-main-text">{{ score }}</h1>
                </div>
            </div>
            
            <h2 id="question">{{ currentQuestion.question }}</h2>
            
            <div 
                v-for="choiceNum in 4" 
                :key="choiceNum" 
                class="choice-container" 
                @click="selectChoice(choiceNum)"
                :class="{
                    correct: choiceNum === currentQuestion.answer && !acceptingAnswers && selectedAnswer === choiceNum,
                    incorrect: selectedAnswer === choiceNum && selectedAnswer !== currentQuestion.answer && !acceptingAnswers
                }"
            >
                <p class="choice-prefix">{{ String.fromCharCode(64 + choiceNum) }}</p>
                <p class="choice-text" :data-number="choiceNum">{{ currentQuestion['choice' + choiceNum] }}</p>
            </div>
        </div>
        
        <div v-if="currentScreen === 'end'" id="end-game-screen" class="screen flex-center flex-column">
            <h1 class="end-message">Quiz Complete!</h1>
            <h2 id="final-score">{{ score }}</h2>
            <div class="end-buttons">
                <button :disabled="saveBtnDisabled" @click="saveScore">{{ saveBtnText }}</button>
                <button @click="startQuiz">Play Again</button>
                <button @click="showHighScoresFromEnd">High Scores</button>
                <button @click="goToNextQuiz">Next Quiz</button>
                <button @click="showHome">Home</button>
            </div>
        </div>
        
        <div v-if="currentScreen === 'highscore'" id="highscore-screen" class="screen flex-center flex-column">
            <h1 class="end-message">High Scores</h1>
            <div id="highscore-list">
                <div v-if="highScores.length === 0" class="no-scores">No high scores yet. Play a game and save your score!</div>
                <div v-else v-for="(scoreObj, index) in highScores" :key="index" class="highscore-item">
                    <div>
                        <span class="highscore-name">#{{ index + 1 }} {{ scoreObj.name }}</span>
                        <small style="display: block; color: #666; margin-top: 5px;">{{ scoreObj.date }} at {{ scoreObj.time || 'Unknown time' }}</small>
                    </div>
                    <span class="highscore-score">{{ scoreObj.score }}</span>
                </div>
            </div>
            <div class="end-buttons">
                <button @click="backFromHighScores">Back</button>
                <button @click="showHome">Home</button>
            </div>
        </div>
    </div>
</template>

<script setup>
import { ref, reactive, onMounted } from 'vue';

// Reactive state variables
const currentScreen = ref('home'); // 'home', 'game', 'end', 'highscore'
const currentQuestion = ref({});
const acceptingAnswers = ref(false);
const score = ref(0);
const questionCounter = ref(0);
const availableQuestions = reactive([]); // Use reactive for arrays/objects if not changing the reference
const previousScreen = ref('home'); // To know where to go back from high scores

const highScores = reactive([]); // Will store parsed high scores

// Game-specific constants
const CORRECT_BONUS = 10;
const MAX_QUESTIONS = 10;

const questions = [
    {
        question: "Inside which HTML element do we put the Javascript?",
        choice1: "<script>",
        choice2: "<javascript>",
        choice3: "<js>",
        choice4: "<scripting>",
        answer: 1
    },
    {
        question: "What is the correct syntax for referring to an external script called 'xxx.js'?",
        choice1: "<script href='xxx.js'>",
        choice2: "<script name='xxx.js'>",
        choice3: "<script src='xxx.js'>",
        choice4: "<script file='xxx.js'>",
        answer: 3
    },
    {
        question: "What is the correct syntax for referring to an external style called 'xxx.css'?",
        choice1: "<link file='xxx.css'>",
        choice2: "<link name='xxx.css'>",
        choice3: "<link src='xxx.css'>",
        choice4: "<link href='xxx.css'>",
        answer: 4
    },
    {
        question: "JavaScript is a _______________ language.",
        choice1: "Object-Oriented",
        choice2: "High-level",
        choice3: "Assembly-language",
        choice4: "Object-Based",
        answer: 4
    },
    {
        question: "Initialization of variables can be done by writing which operator between variable name and operand value?",
        choice1: "EQUALS",
        choice2: "=",
        choice3: "VALUE",
        choice4: "==",
        answer: 2
    },
    {
        question: "Which of the following keywords is used to define a variable in JavaScript?",
        choice1: "var",
        choice2: "let",
        choice3: "Both A and B",
        choice4: "None of the above",
        answer: 3
    },
    {
        question: "Variable Scope determines the accessibility (visibility) of variables.",
        choice1: "TRUE",
        choice2: "FALSE",
        choice3: "Sometimes",
        choice4: "Depends on browser",
        answer: 1
    },
    {
        question: "A variable stores the data value that can be changed later.",
        choice1: "TRUE",
        choice2: "FALSE",
        choice3: "Only for strings",
        choice4: "Only for numbers",
        answer: 1
    },
    {
        question: "Any variable declared inside a block such as a function can be accessed anywhere in a program.",
        choice1: "TRUE",
        choice2: "FALSE",
        choice3: "Only with var",
        choice4: "Only with let",
        answer: 2
    },
    {
        question: "A named function definition executes automatically.",
        choice1: "TRUE",
        choice2: "FALSE",
        choice3: "Sometimes",
        choice4: "Only in strict mode",
        answer: 2
    }
];

// UI state for save button
const saveBtnText = ref("Save Score");
const saveBtnDisabled = ref(false);
const selectedAnswer = ref(null); // To highlight correct/incorrect choice

// --- Screen Navigation Functions ---
const showHome = () => {
    currentScreen.value = 'home';
};

const startQuiz = () => {
    previousScreen.value = 'home';
    startGame();
};

const showHighScoresFromHome = () => {
    previousScreen.value = 'home';
    showHighScores();
};

const showHighScoresFromEnd = () => {
    previousScreen.value = 'end';
    showHighScores();
};

// --- Game Logic Functions ---
const startGame = () => {
    questionCounter.value = 0;
    score.value = 0;
    // Deep copy the questions array to availableQuestions for shuffling
    availableQuestions.splice(0, availableQuestions.length, ...JSON.parse(JSON.stringify(questions)));
    currentScreen.value = 'game';
    getNewQuestion();
};

const getNewQuestion = () => {
    questionCounter.value++;
    selectedAnswer.value = null; // Reset selected answer highlight

    if (availableQuestions.length === 0 || questionCounter.value > MAX_QUESTIONS) {
        return endGame();
    }

    const questionIndex = Math.floor(Math.random() * availableQuestions.length);
    currentQuestion.value = availableQuestions[questionIndex];

    availableQuestions.splice(questionIndex, 1); // Remove question from available
    acceptingAnswers.value = true;
};

const selectChoice = (choiceNum) => {
    if (!acceptingAnswers.value) return;

    acceptingAnswers.value = false;
    selectedAnswer.value = choiceNum; // Set selected answer for highlighting
    
    const classToApply = choiceNum === currentQuestion.value.answer ? "correct" : "incorrect";
    
    if (classToApply === "correct") {
        score.value += CORRECT_BONUS;
    }
    
    setTimeout(() => {
        getNewQuestion();
    }, 1000);
};

const endGame = () => {
    currentScreen.value = 'end';
    saveBtnText.value = "Save Score"; // Reset save button text
    saveBtnDisabled.value = false; // Enable save button
};

const goToNextQuiz = () => {
    alert("Next Quiz feature coming soon!\n\nThis could:\n• Load a different set of questions\n• Navigate to another quiz page\n• Show advanced level questions\n• Connect to a quiz series");
};

// --- High Score Management ---
const saveScore = () => {
    const name = prompt("Enter your name:");
    if (name && name.trim()) {
        const currentHighScores = getHighScoresFromLocalStorage(); // Get latest from storage
        const newScore = {
            name: name.trim(),
            score: score.value,
            date: new Date().toLocaleDateString(),
            time: new Date().toLocaleTimeString()
        };
        
        currentHighScores.push(newScore);
        currentHighScores.sort((a, b) => b.score - a.score);
        
        if (currentHighScores.length > 10) {
            currentHighScores.splice(10);
        }
        
        localStorage.setItem('gameHighScores', JSON.stringify(currentHighScores));
        
        saveBtnText.value = "✓ Saved!";
        saveBtnDisabled.value = true;
        
        const ranking = currentHighScores.findIndex(s => 
            s.name === name.trim() && 
            s.score === score.value && 
            s.date === newScore.date && 
            s.time === newScore.time
        ) + 1;
        
        alert(`Score saved successfully!\nYour score: ${score.value}\nRanking: #${ranking} out of ${currentHighScores.length}`);
        
        // Update reactive highScores array for display
        highScores.splice(0, highScores.length, ...currentHighScores);

        setTimeout(() => {
            previousScreen.value = 'end';
            showHighScores();
        }, 1000);
    }
};

const getHighScoresFromLocalStorage = () => {
    const storedHighScores = localStorage.getItem('gameHighScores');
    return storedHighScores ? JSON.parse(storedHighScores) : [];
};

const showHighScores = () => {
    highScores.splice(0, highScores.length, ...getHighScoresFromLocalStorage()); // Populate reactive array
    currentScreen.value = 'highscore';
};

const backFromHighScores = () => {
    if (previousScreen.value === 'home') {
        showHome();
    } else {
        currentScreen.value = 'end'; // Go back to end screen
    }
};

// --- Lifecycle Hook ---
// This runs once the component is mounted (added to the DOM)
onMounted(() => {
    // Optionally load high scores on initial mount if you want to display them immediately
    // or just let the showHighScores function handle it when triggered.
    // getHighScoresFromLocalStorage(); 
});
</script>
