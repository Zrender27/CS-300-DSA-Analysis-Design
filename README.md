# CS-300-DSA-Analysis-Design
In this course, I worked on solving problems related to organizing and managing large sets of data efficiently using different data structures. Specifically, my project focused on building a Bid Processing System that could load, search, insert, and remove bids from a dataset. The challenge was not only to make the program functional but also to ensure that operations were optimized for performance depending on which data structure—such as a linked list, hash table, or binary search tree—was used.
My approach began with analyzing the core problem: how to store and retrieve data in the most efficient way possible. Understanding data structures was essential because each one provides unique trade-offs between time and space complexity. For example, linked lists make insertions simple but are slow for searches, while binary search trees allow fast searches if balanced. By comparing these structures, I was able to determine which implementation best met the project’s requirements.
Throughout the project, I encountered several roadblocks—such as handling pointer errors, debugging tree traversal logic, and ensuring data was loaded correctly from CSV files. I overcame these challenges by incrementally testing functions, reviewing algorithm flow step-by-step, and using debugging output to trace data through my code. Consulting course materials and applying logical reasoning helped me identify where my assumptions about data flow were incorrect and fix them systematically.
Completing this project has expanded my approach to software design and program development. I now design with scalability and clarity in mind—structuring programs into smaller, reusable components and selecting data structures deliberately based on the problem’s needs rather than by habit. It reinforced that good software design is about balancing performance, clarity, and adaptability.
This project also evolved the way I write code that is maintainable, readable, and adaptable. I improved my use of consistent naming conventions, inline documentation, and modular design. I now think ahead about how another developer—or my future self—might modify or extend the program. I’ve learned to prioritize clean structure, clear logic, and efficiency equally, which will continue to guide how I approach software engineering challenges going forward.




[ProjectTwo.cpp](https://github.com/user-attachments/files/22994510/ProjectTwo.cpp)
/*
  ProjectTwo.cpp
  Zoe Render — October 19, 2025

  ABCU Advising Assistance Program (CS 300 Project Two)
  -----------------------------------------------------
  • Reads a CSV file of courses: COURSE_NUM, TITLE, [PREREQ_1, PREREQ_2, ...]
  • Stores Course objects in a Binary Search Tree keyed by courseNum
  • Menu:
      1) Load Data Structure
      2) Print Course List (alphanumeric)
      3) Print Course (title + prerequisite numbers AND titles)
      9) Exit
  • Single-file, no external headers.

  Notes:
  - Input is case-insensitive for course lookups; course numbers are normalized to UPPERCASE.
  - File validation:
      * Each line must contain at least courseNum and title.
      * Every listed prerequisite must also exist as a course in the file.
*/

#include <iostream>
#include <fstream>
#include <sstream>
#include <string>
#include <vector>
#include <cctype>
#include <algorithm>
#include <unordered_set>

using namespace std;

// ------------ Utilities ------------
static inline string trim(string s) {
    // remove leading
    size_t start = s.find_first_not_of(" \t\r\n");
    if (start == string::npos) return "";
    size_t end = s.find_last_not_of(" \t\r\n");
    return s.substr(start, end - start + 1);
}

static inline string upper(string s) {
    for (char& c : s) c = static_cast<char>(toupper(static_cast<unsigned char>(c)));
    return s;
}

static vector<string> splitCSVLine(const string& line) {
    // Simple CSV split by comma with no quoted-field requirements for this dataset
    vector<string> tokens;
    string token;
    istringstream ss(line);
    while (getline(ss, token, ',')) {
        tokens.push_back(trim(token));
    }
    // ignore trailing empties
    while (!tokens.empty() && tokens.back().empty()) tokens.pop_back();
    return tokens;
}

// ------------ Domain Model ------------
struct Course {
    string courseNum;           // e.g., "CSCI200"
    string title;               // e.g., "Data Structures"
    vector<string> prereqs;     // list of prerequisite course numbers (UPPERCASE)
};

// ------------ BST ------------
struct Node {
    Course data;
    Node* left;
    Node* right;
    explicit Node(const Course& c) : data(c), left(nullptr), right(nullptr) {}
};

class BST {
public:
    BST() : root(nullptr) {}
    ~BST() { destroy(root); }

    void insert(const Course& c) {
        if (!root) { root = new Node(c); return; }
        addNode(root, c);
    }

    // Returns pointer to found Course, else nullptr (do not free the pointer!)
    const Course* search(const string& key) const {
        Node* cur = root;
        string k = upper(key);
        while (cur) {
            if (k == cur->data.courseNum) return &cur->data;
            cur = (k < cur->data.courseNum) ? cur->left : cur->right;
        }
        return nullptr;
    }

    // In-order print (alphanumeric by courseNum)
    void printAll() const { inOrder(root); }

private:
    Node* root;

    static void destroy(Node* n) {
        if (!n) return;
        destroy(n->left);
        destroy(n->right);
        delete n;
    }

    static void addNode(Node* node, const Course& c) {
        if (c.courseNum < node->data.courseNum) {
            if (!node->left) node->left = new Node(c);
            else addNode(node->left, c);
        } else if (c.courseNum > node->data.courseNum) {
            if (!node->right) node->right = new Node(c);
            else addNode(node->right, c);
        } else {
            // duplicate key -> replace payload to keep latest info
            node->data = c;
        }
    }

    static void printCourseLine(const Course& c) {
        cout << c.courseNum << ", " << c.title << '\n';
    }

    static void inOrder(Node* node) {
        if (!node) return;
        inOrder(node->left);
        printCourseLine(node->data);
        inOrder(node->right);
    }
};

// ------------ Loading & Validation ------------

struct LoadResult {
    bool ok{false};
    string message;
    vector<Course> courses; // parsed and validated
};

/*
  Two-pass load:
  Pass 1: parse lines into tokens, collect course numbers (for prereq validation)
  Pass 2: build Course objects and validate that each prereq exists
*/
LoadResult parseAndValidateFile(const string& path) {
    LoadResult res;
    ifstream in(path);
    if (!in.is_open()) {
        res.message = "Error: could not open file: " + path;
        return res;
    }

    vector<vector<string>> raw;
    unordered_set<string> known;
    string line;
    size_t lineNum = 0;

    // Read all lines
    while (getline(in, line)) {
        ++lineNum;
        line = trim(line);
        if (line.empty()) continue; // skip blank lines
        auto tokens = splitCSVLine(line);

        if (tokens.size() < 2) {
            res.message = "Format error at line " + to_string(lineNum) +
                          ": expected at least COURSE_NUM,TITLE";
            return res;
        }
        // Normalize course number to UPPERCASE; title kept as-is
        tokens[0] = upper(tokens[0]);
        raw.push_back(tokens);
        known.insert(tokens[0]);
    }
    in.close();

    if (raw.empty()) {
        res.message = "Error: file is empty.";
        return res;
    }

    // Build Courses and validate prerequisites
    vector<Course> courses;
    courses.reserve(raw.size());
    for (const auto& tokens : raw) {
        Course c;
        c.courseNum = tokens[0];
        c.title     = tokens[1];
        for (size_t i = 2; i < tokens.size(); ++i) {
            string p = upper(tokens[i]);
            if (!p.empty()) {
                if (known.find(p) == known.end()) {
                    res.message = "Error: prerequisite '" + p + "' listed for " + c.courseNum +
                                  " does not exist in file.";
                    return res;
                }
                c.prereqs.push_back(p);
            }
        }
        courses.push_back(std::move(c));
    }

    res.ok = true;
    res.message = "Loaded " + to_string(courses.size()) + " courses.";
    res.courses = std::move(courses);
    return res;
}

// Load into BST after validation
bool loadIntoBST(const string& path, BST& bst, string& statusMsg) {
    auto parsed = parseAndValidateFile(path);
    statusMsg = parsed.message;
    if (!parsed.ok) return false;

    for (const auto& c : parsed.courses) {
        bst.insert(c);
    }
    return true;
}

// Print one course with prerequisite numbers AND titles
void printCourseWithPrereqs(const BST& bst, const string& inputKey) {
    string key = upper(trim(inputKey));
    const Course* c = bst.search(key);
    if (!c) {
        cout << "Course not found.\n";
        return;
    }

    cout << c->courseNum << ", " << c->title << '\n';
    if (c->prereqs.empty()) {
        cout << "Prerequisites: None\n";
        return;
    }

    cout << "Prerequisites: ";
    for (size_t i = 0; i < c->prereqs.size(); ++i) {
        const string& pn = c->prereqs[i];
        const Course* pc = bst.search(pn);
        if (pc) {
            cout << pc->courseNum << " (" << pc->title << ")";
        } else {
            // Should not happen if validation succeeded, but be defensive
            cout << pn;
        }
        if (i + 1 < c->prereqs.size()) cout << ", ";
    }
    cout << '\n';
}

// ------------ Program ------------
int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);

    BST bst;
    bool loaded = false;
    string loadedFile;

    cout << "Welcome to the course planner.\n\n";

    while (true) {
        cout << "  1. Load Data Structure.\n";
        cout << "  2. Print Course List.\n";
        cout << "  3. Print Course.\n";
        cout << "  9. Exit\n\n";
        cout << "What would you like to do? ";

        string choice;
        if (!getline(cin, choice)) break;
        choice = trim(choice);

        if (choice == "1") {
            cout << "Enter file name: ";
            string path;
            if (!getline(cin, path)) break;
            path = trim(path);

            string msg;
            loaded = loadIntoBST(path, bst, msg);
            loadedFile = loaded ? path : "";
            cout << msg << "\n\n";

        } else if (choice == "2") {
            if (!loaded) {
                cout << "Please load the data structure first (option 1).\n\n";
                continue;
            }
            cout << "Here is a sample schedule:\n\n";
            bst.printAll();
            cout << '\n';

        } else if (choice == "3") {
            if (!loaded) {
                cout << "Please load the data structure first (option 1).\n\n";
                continue;
            }
            cout << "What course do you want to know about? ";
            string key;
            if (!getline(cin, key)) break;
            key = trim(key);
            printCourseWithPrereqs(bst, key);
            cout << '\n';

        } else if (choice == "9") {
            cout << "Thank you for using the course planner!\n";
            break;

        } else if (!choice.empty()) {
            cout << choice << " is not a valid option.\n\n";
        }
    }
    return 0;
}

