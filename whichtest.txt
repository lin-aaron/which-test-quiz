function Question(questionText, answerText, reason) {
    this.questionText = questionText;
    this.answerText = answerText;
    this.reason = reason;
    this.userAnswer = false;
}

function Quiz(questions, type, triggers) {
    this.questions = questions;
    this.type = type;
    this.triggers = triggers;
    this.testRecs = [false, false];
    this.answered = [];

    this.calculateRisk = function() {
        if (this.answered) {
            risk = 0;
            var i = 0;
            do {
                if (this.triggers[i].triggered) {
                    risk = this.triggers[i].risk;
                }
                i++;
            } while (!this.triggers[i - 1].triggered && i < this.triggers.length);

            return risk;
        }
    }

    this.getRecTest = function() {
        var tests = [];
        var i = 0;
        do {
            if (this.triggers[i].triggered) {
                tests = tests.concat(this.triggers[i].tests);
            }
            i++;
        } while (!this.triggers[i - 1].triggered && i < this.triggers.length);
        return tests;
    }
}
// Trigger object contains all the triggers that recommend a given set of tests
// Example: new Trigger(1, [0b11, 0b100], 2, [0, 4]);
//     -Water source is PWS-Large (0)
//     -0b11 corresponds to positive answers to first two questions
//      0b100 corresponds to negative answers to first two, positive to third
//     -Both result in a high risk (2) recommendation (ranges from 0 to 2)
//     -and cause the quiz to recommend the 0th and 4th tests in 'tests'
function Trigger(type, sequences, risk, tests, triggered = false) {
    this.type = type;
    this.sequences = sequences;
    this.risk = risk;
    this.tests = tests;
    this.triggered = triggered;

    this.update = function(answer) {
        function match(sequence) {
            var bitmask = sequence ? 1 : 0;
            return (trigger & 1) == (bitmask);
        }

        for (var i = 0; i < this.sequences.length; i++) {
            var trigger = this.sequences[i];
            if (match(answer)) { //if it matches, shift over one
                this.sequences[i] = this.sequences[i] >> 1;
                if (this.sequences[i] == 0) { //if it's zero, mark as triggered
                    this.triggered = true;
                    this.sequences.splice(i, 1);
                    i--;
                }
            } else { //otherwise, delete it
                this.sequences.splice(i, 1);
                i--;
            }
        }
    }
}

var questionOrder = [
    [0, 1, 2, 3, 4], //Well
    [4, 3, 2, 0, 1], //PWS-Large
    [4, 3, 2, 1, 0]  //PWS-Small
]; 

var triggers = [];

function setTriggers(){
    triggers = [
        [ //Well triggers
            new Trigger(0, [0b11, 0b101, 0b1001, 0b10001, 0b110, 0b1010, 0b11000], 2, [0]), //CC
            new Trigger(0, [0b1], 3, [0]), //CC with non recent message
            new Trigger(0, [0b10100], 0, [0, 3]), //CC + lead (low risk exception)
            new Trigger(0, [999], 0, [3], true) //Default no match
        ],
        [ //PWS-Large triggers
            new Trigger(1, [0b111, 0b1011, 0b10110], 1, [4, 1]), //CC
            new Trigger(1, [0b11, 0b1001, 0b110, 0b11100], 1, [4]), //EC
            new Trigger(1, [999], 0, [3], true) //Default no match
        ],
        [ //PWS-Small triggers
            new Trigger(2, [0b1110, 0b11010, 0b10110, 0b11100], 2, [1]), //CC
            new Trigger(2, [0b101, 0b10101, 0b11, 0b10010, 0b10001, 0b10100], 1, [4]), //EC
            new Trigger(2, [999], 0, [3], true) //Default no match
        ]
    ]
}
var tests = [
    { name: 'Advanced Well Water Test', link: 'https://mytapscore.com/products/simple-water-home-test', img: 'https://cdn.shopify.com/s/files/1/1235/7388/files/Tapscore-Well-Owner-Test_MQ.png?15623486734411390152', description: 'For the most common US groundwater contaminants. The best way to protect your well.' },
    { name: 'Advanced City Water Test', link: 'https://mytapscore.com/products/the-public-water-supply-test', img: 'https://cdn.shopify.com/s/files/1/1235/7388/files/Tapscore-Utility-Customer-Test_MQ.png?15623486734411390152', description: 'Adds Volatile Organic Compounds and Disinfection Byproducts to the Essential City Water Test.' },
    { name: '"Top Hazards" Heavy Metals Test', link: 'https://mytapscore.com/products/heavy-metals-hardness-test', img: 'Tapscore-Heavy-Metals-Test_LQ.png', description: 'They come from old pipes, natural deposits, and industry. Common and dangerous. Includes lead & copper.' },
    { name: 'Lead & Copper Test', link: 'https://mytapscore.com/products/the-lead-test', img: 'https://cdn.shopify.com/s/files/1/1235/7388/files/Tapscore-Lead-Test_MQ.png?12019801014422657681', description: 'The most common metals in US drinking water. Lead is a neurotoxin that becomes dangerous at levels far below what test strips detect.' },
    { name: 'Essential City Water Test', link: 'https://mytapscore.com/products/essential-city-water-test', img: 'https://cdn.shopify.com/s/files/1/1235/7388/files/Tapscore-Heavy-Metals-Test_MQ.png?15623486734411390152', description: 'Tests contaminants that appear after water leaves the utility, and a few they don\'t remove completely.' },
];

var resultText = {
    open: ["Based on your responses your drinking water is likely not at high risk for contamination. You may still want to be proactive in testing your water for the most common heavy metals in US drinking water, which can show up in unexpected places.",
        "The following factors suggest your drinking water may have an elevated risk from contamination:",
        "The following factors suggest your drinking water may have a high risk from contamination:",
        "The following factors suggest your drinking water may have a high risk from contamination:"
    ],
    close: ["",
        "Because of this, we recommend the following Tap Score package to check your water for heavy metals from old pipes or other sources",
        "Because of this, we recommend the following Tap Score package to reveal what's in your water",
        "If you've never tested your well water then we highly recommend our advanced well testing package to establish a baseline for your well's water quality, especially if you use it for drinking or cooking"
    ]
}

var questions = [new Question("Have you performed a complete test of your water within the last 3 years and found nothing potentially hazardous?", ["NO", "YES"], "You haven't performed a complete test recently."),
    new Question("Do you live near any significant industrial or agricultural activity?", ["YES", "NO"], "You live near contaminant emitters (industry or agriculture)."),
    new Question("Have you recently noticed any odd smell or taste coming from your water, or a change in color? Have you seen recent news about local drinking water incidents?", ["YES", "NO"], "You've noticed a recent change in your water's characteristics or heard about local water incidents."),
    new Question("Do infants or young children live in your home? Will they in the near future? Select yes if any other vulnerable individuals live in the home, such as the elderly or chronically ill.", ["YES", "NO"], "There are or will soon be vulnerable individuals in your home"),
    new Question("When was your home built?", ["Before 1986", "After 1986"], "Your home was built before lead pipes were completely outlawed.")
];

var types = ['Well', 'Water Utility'];

var almostCounter = 0;

function startQuiz() {
    almostCounter = 0;
    var start = $('<div>', { id: 'which-test-start' }); //make start div
    $('#which-test-restart', '.which-test-progBar').remove();
    start.append($('<h1 class="which-test-title">Not sure which is right for you?<br>Figure it out in 20 seconds with this quiz!</h1>'));
    start.append($('<button id="which-test-start-button">TAKE THE QUIZ</button>'));
    $('#which-test-container').addClass('which-test-text');
    $('#which-test-container').prepend(start).fadeIn();
}

function createQuiz(type) { //Orders quiz questions 
    var orderedQuestions = [];
    for (var i = 0; i < questionOrder[type].length; i++) {
        orderedQuestions[i] = questions[questionOrder[type][i]];
    }   
    setTriggers();
    quiz = new Quiz(orderedQuestions, type, triggers[type]); //sets global quiz to a new quiz
}

function createQuestion() { //returns the next question
    var currQ = quiz.questions.shift();
    var question = $('<div>', { id: 'which-test-question' });
    question.append($('<h2 id="question-text">' + currQ.questionText + '</h2>'));
    question.append(createAnswers(currQ));
    quiz.answered.push(currQ);
    return question;
}

function createAnswers(question) {
    var row = $('<ul>', { id: 'which-test-answers', class: 'which-test-ul' });
    row.append($('<li>', { id: 'which-test-true', class: 'which-test-answer', text: question.answerText[0] }));
    row.append($('<li>', { id: 'which-test-false', class: 'which-test-answer', text: question.answerText[1] }));
    return row;
}

function displayAlmostDone() { //displays an almost done page
    // $('.which-test-progBar-bkgd').fadeOut(function(){
    $('#which-test-quiz').fadeOut(function() {
        $('#which-test-question').remove();
        $('#which-test-restart').remove();
        $('.which-test-progBar').remove();
        $('.which-test-almost').remove();
        var almost = $('<div id="which-test-almost">Great! Just a few more questions.</div>');
        $('#which-test-quiz').append(almost);
        $('#which-test-quiz, .which-test-progBar').fadeIn();
    });
    // });
}

function createProgressBar() {
    var bar = $('<div>', { class: 'which-test-progBar which-test-progBar-bkgd' });
    bar.append($('<div>', { class: 'which-test-progBar which-test-progBar-fill' }));
    return bar;
}

function displayNext() { //displays the almost done page, waits 1.0 seconds, then shows the next question
    if (almostCounter == 2) {
        displayAlmostDone();
    }
    window.setTimeout(function() {
        var question = createQuestion();
        $('#which-test-quiz').fadeOut(function() {
            $('#which-test-almost, #which-test-question, #which-test-restart, .which-test-progBar').remove();
            $('#which-test-quiz').append(question).append($('<p>', { id: 'which-test-restart', text: 'Start Over' }));
            $('#which-test-quiz').prepend(createProgressBar());
            // console.log(quiz);
            var length = 20 * (quiz.answered.length - 1);
            $('.which-test-progBar-fill').width(length + '%');
            $('#which-test-quiz').fadeIn();
            recorded = false;
            almostCounter++;
        })
    }, 1000 * (almostCounter == 2 ? 1 : 0))
}

function recordAnswer(ans) {
    quiz.answered[quiz.answered.length - 1].userAnswer = ans;
    quiz.triggers.forEach(function(trigger) {
        console.log(trigger)
        trigger.update(ans);
    });
    console.log(quiz);
}

function createResults() {
    //calculate risk
    quiz.risk = quiz.calculateRisk();

    var totalResults = $('<div>', { id: 'which-test-results' });
    totalResults.append($('<p>', { id: 'which-test-open', text: resultText.open[quiz.risk] }));
    totalResults.append(createReasonList());
    if (quiz.risk > 0) {
        totalResults.append($('<p>', { id: 'which-test-close', text: resultText.close[quiz.risk] + ':' }));
    }
    return totalResults;
}

function createReasonList() {
    if (quiz.risk > 0) {
        var reasons = $('<ul>', { id: 'which-test-reasons', class: 'which-test-ul' });
        for (var i = 0; i < quiz.answered.length; i++) {
            if (quiz.answered[i].userAnswer) {
                var reason = $('<li>', { class: 'which-test-reason', text: '- ' + quiz.answered[i].reason });
                reasons.append(reason);
            }
        }
    }
    return reasons;
}

function displayResults() {
    var quizDiv = $('#which-test-quiz');
    quizDiv.fadeOut(function() {
        $('#which-test-question').remove();
        $('.which-test-progBar').remove();
        var results = createResults();
        if (type >= 1) { type = 1; }
        var testNums = quiz.getRecTest();
        console.log(testNums);
        for (var i = testNums.length - 1; i >= 0; i--) {
            quizDiv.prepend(createTest(tests[testNums[i]]));
        }
        quizDiv.prepend(results);
        recorded = false;
        quizDiv.fadeIn();
    });

}

function createTest(testObj) {
    console.log(testObj);
    var test = $('<div>', { class: 'test' });
    var img = $('<img>', { class: 'test-pic', src: testObj.img, alt: 'Water Test Package' });
    var title = $('<p>', { class: 'test-title', text: testObj.name });
    var description = $('<p>', { class: 'test-description', text: testObj.description });
    var text = $('<div>', { class: 'test-container' });
    text.append(title).append(description);
    test.append(img).append(text);
    return $('<a>', { class: 'test', href: testObj.link }).append(test);
}

$(document).on('click', '#which-test-start-button', function(e) { // start test, ask for water source
    e.preventDefault();
    var askType = $('<div>', { id: 'which-test-question' });
    askType.append($('<h2>First, tell us where the water you want to test comes from.</ h2>'));
    var choices = $('<ul>', { id: 'which-test-answers', class: 'which-test-ul' });
    choices.append($('<li>', { id: 'well', class: 'which-test-answer', text: 'A private or shared groundwater well' }));
    choices.append($('<li>', { id: 'pwsl', class: 'which-test-answer', text: 'A water utility in a large city' }));
    choices.append($('<li>', { id: 'pwss', class: 'which-test-answer', text: 'A water utility in a small town' }));
    askType.append(choices);
    $('#which-test-container > *').fadeOut(function() {
        $('#which-test-container').prepend($('<div>', { id: 'which-test-quiz' }));
        $('#which-test-quiz').append(askType);
        $('#which-test-start').remove();
        $('#which-test-quiz').fadeIn();
    });
});

$(document).on('click', '#well', function(e) { //well
    e.preventDefault();
    type = 0;
    createQuiz(type);
    displayNext();
});

$(document).on('click', '#pwsl', function(e) { //pswl
    e.preventDefault();
    type = 1;
    createQuiz(type);
    displayNext();
});

$(document).on('click', '#pwss', function(e) { //pwss
    e.preventDefault();
    type = 2;
    createQuiz(type);
    displayNext();
});

$(document).on('click', '#which-test-true', function(e) { //true answer
    e.preventDefault();
    if (!recorded) {
        recorded = true;
        recordAnswer(true);
        if (quiz.questions.length == 0) {
            displayResults();
        } else {
            displayNext();
        }
    }
});

$(document).on('click', '#which-test-false', function(e) { //false answer
    e.preventDefault();
    if (!recorded) {
        recorded = true;
        recordAnswer(false);
        if (quiz.questions.length == 0) {
            displayResults();
        } else {
            displayNext();
        }
    }
});

$(document).on('click', '#which-test-restart', function(e) { //restart button
    e.preventDefault();
    quiz = null;
    $('#which-test-quiz').fadeOut(function() {
        $('#which-test-quiz').remove();
        startQuiz();
    });
});

startQuiz();