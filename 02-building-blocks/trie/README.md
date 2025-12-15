# Trie (Prefix Tree)

## Overview

A **Trie** (pronounced "try") is a tree-based data structure for storing strings efficiently. It enables fast prefix-based searches and is commonly used for autocomplete, spell checking, and IP routing.

## Key Properties

- **Root**: Empty node
- **Edges**: Labeled with characters
- **Paths**: Root to node represents a string
- **End markers**: Flag to indicate complete words

## Time Complexity

| Operation | Time | Space |
|-----------|------|-------|
| Insert | O(m) | O(m) |
| Search | O(m) | - |
| StartsWith | O(m) | - |
| Delete | O(m) | - |

where m = length of string

## Implementation

```python
class TrieNode:
    """Node in Trie"""

    def __init__(self):
        self.children = {}  # char -> TrieNode
        self.is_end_of_word = False
        self.word_count = 0  # Number of words ending here

class Trie:
    """
    Prefix tree for efficient string storage and retrieval

    Use cases:
    - Autocomplete
    - Spell checker
    - IP routing (longest prefix match)
    - Dictionary
    """

    def __init__(self):
        self.root = TrieNode()

    def insert(self, word: str) -> None:
        """
        Insert word into trie

        Time: O(m) where m = len(word)
        Space: O(m) worst case (no shared prefixes)
        """
        node = self.root

        for char in word:
            if char not in node.children:
                node.children[char] = TrieNode()
            node = node.children[char]

        node.is_end_of_word = True
        node.word_count += 1

    def search(self, word: str) -> bool:
        """
        Search for exact word in trie

        Time: O(m)
        """
        node = self._find_node(word)
        return node is not None and node.is_end_of_word

    def starts_with(self, prefix: str) -> bool:
        """
        Check if any word starts with prefix

        Time: O(m)
        """
        return self._find_node(prefix) is not None

    def _find_node(self, prefix: str) -> TrieNode:
        """Helper: find node corresponding to prefix"""
        node = self.root

        for char in prefix:
            if char not in node.children:
                return None
            node = node.children[char]

        return node

    def delete(self, word: str) -> bool:
        """
        Delete word from trie

        Returns True if deleted, False if not found
        """
        def _delete_helper(node, word, index):
            if index == len(word):
                # Reached end of word
                if not node.is_end_of_word:
                    return False  # Word doesn't exist

                node.is_end_of_word = False

                # Can delete node if no children
                return len(node.children) == 0

            char = word[index]
            if char not in node.children:
                return False  # Word doesn't exist

            child = node.children[char]
            should_delete_child = _delete_helper(child, word, index + 1)

            if should_delete_child:
                del node.children[char]
                # Can delete current node if:
                # 1. Not end of another word
                # 2. No other children
                return not node.is_end_of_word and len(node.children) == 0

            return False

        return _delete_helper(self.root, word, 0)

    def get_all_words(self) -> list:
        """Return all words in trie"""
        words = []

        def dfs(node, current_word):
            if node.is_end_of_word:
                words.append(current_word)

            for char, child in node.children.items():
                dfs(child, current_word + char)

        dfs(self.root, "")
        return words

    def get_words_with_prefix(self, prefix: str) -> list:
        """
        Get all words starting with prefix

        Use case: Autocomplete
        """
        node = self._find_node(prefix)
        if node is None:
            return []

        words = []

        def dfs(node, current_word):
            if node.is_end_of_word:
                words.append(current_word)

            for char, child in node.children.items():
                dfs(child, current_word + char)

        dfs(node, prefix)
        return words

# Example Usage
if __name__ == '__main__':
    trie = Trie()

    # Insert words
    words = ["apple", "app", "application", "apply", "banana"]
    for word in words:
        trie.insert(word)

    # Search
    print(trie.search("app"))  # True
    print(trie.search("appl"))  # False (prefix but not complete word)

    # Starts with (prefix search)
    print(trie.starts_with("app"))  # True
    print(trie.starts_with("ban"))  # True

    # Autocomplete
    suggestions = trie.get_words_with_prefix("app")
    print(suggestions)  # ['app', 'apple', 'application', 'apply']

    # Delete
    trie.delete("app")
    print(trie.search("app"))  # False
    print(trie.search("apple"))  # True (not affected)

    # All words
    print(trie.get_all_words())
```

## Advanced Features

### 1. Word Frequency Tracking

```python
class TrieWithFrequency(Trie):
    """Trie that tracks word frequencies"""

    def insert(self, word: str) -> None:
        node = self.root
        for char in word:
            if char not in node.children:
                node.children[char] = TrieNode()
            node = node.children[char]

        node.is_end_of_word = True
        node.word_count += 1

    def get_frequency(self, word: str) -> int:
        """Get frequency of word"""
        node = self._find_node(word)
        if node and node.is_end_of_word:
            return node.word_count
        return 0

    def get_top_k_suggestions(self, prefix: str, k: int) -> list:
        """Get top K most frequent words with prefix"""
        node = self._find_node(prefix)
        if not node:
            return []

        candidates = []

        def dfs(node, current_word):
            if node.is_end_of_word:
                candidates.append((current_word, node.word_count))

            for char, child in node.children.items():
                dfs(child, current_word + char)

        dfs(node, prefix)

        # Sort by frequency (descending)
        candidates.sort(key=lambda x: x[1], reverse=True)

        return [word for word, _ in candidates[:k]]
```

### 2. Compressed Trie (Radix Tree)

```python
class RadixNode:
    """Node for compressed trie"""

    def __init__(self, edge_label=""):
        self.edge_label = edge_label  # String on edge
        self.children = {}
        self.is_end_of_word = False

class RadixTree:
    """
    Compressed trie (space-optimized)

    Instead of one char per edge, store strings
    """

    def __init__(self):
        self.root = RadixNode()

    def insert(self, word: str) -> None:
        node = self.root
        i = 0

        while i < len(word):
            char = word[i]

            if char in node.children:
                child = node.children[char]
                label = child.edge_label

                # Find common prefix
                j = 0
                while j < len(label) and i + j < len(word) and label[j] == word[i + j]:
                    j += 1

                if j == len(label):
                    # Exact match of edge label
                    node = child
                    i += j
                else:
                    # Partial match - need to split edge
                    split_node = RadixNode(label[:j])
                    child.edge_label = label[j:]

                    node.children[char] = split_node
                    split_node.children[label[j]] = child

                    if i + j < len(word):
                        # Add remaining part
                        new_child = RadixNode(word[i + j:])
                        new_child.is_end_of_word = True
                        split_node.children[word[i + j]] = new_child
                    else:
                        split_node.is_end_of_word = True

                    return
            else:
                # No child with this character
                new_child = RadixNode(word[i:])
                new_child.is_end_of_word = True
                node.children[char] = new_child
                return

        node.is_end_of_word = True
```

### 3. Ternary Search Tree

```python
class TSTNode:
    """Node for Ternary Search Tree"""

    def __init__(self, char):
        self.char = char
        self.left = None  # Lesser chars
        self.mid = None   # Next char in word
        self.right = None  # Greater chars
        self.is_end_of_word = False

class TernarySearchTree:
    """
    Space-efficient alternative to standard trie

    Each node has 3 children instead of 26 (for alphabet)
    """

    def __init__(self):
        self.root = None

    def insert(self, word: str) -> None:
        def _insert(node, word, index):
            char = word[index]

            if node is None:
                node = TSTNode(char)

            if char < node.char:
                node.left = _insert(node.left, word, index)
            elif char > node.char:
                node.right = _insert(node.right, word, index)
            else:
                if index + 1 < len(word):
                    node.mid = _insert(node.mid, word, index + 1)
                else:
                    node.is_end_of_word = True

            return node

        self.root = _insert(self.root, word, 0)

    def search(self, word: str) -> bool:
        def _search(node, word, index):
            if node is None:
                return False

            char = word[index]

            if char < node.char:
                return _search(node.left, word, index)
            elif char > node.char:
                return _search(node.right, word, index)
            else:
                if index + 1 == len(word):
                    return node.is_end_of_word
                return _search(node.mid, word, index + 1)

        return _search(self.root, word, 0)
```

## Use Cases

### 1. Autocomplete System

```python
class AutocompleteSystem:
    def __init__(self):
        self.trie = TrieWithFrequency()

    def input_character(self, char: str, current_prefix: str) -> list:
        """
        User types a character
        Return top 3 suggestions
        """
        if char == '#':
            # Sentence complete - store it
            self.trie.insert(current_prefix)
            return []

        new_prefix = current_prefix + char
        return self.trie.get_top_k_suggestions(new_prefix, k=3)

# Usage
autocomplete = AutocompleteSystem()
autocomplete.trie.insert("iphone")
autocomplete.trie.insert("ipad")
autocomplete.trie.insert("ipod")
autocomplete.trie.insert("iphone 15")

prefix = ""
for char in "iph":
    suggestions = autocomplete.input_character(char, prefix)
    prefix += char
    print(f"'{prefix}': {suggestions}")

# Output:
# 'i': ['iphone', 'ipad', 'ipod']
# 'ip': ['iphone', 'ipad', 'ipod']
# 'iph': ['iphone', 'iphone 15']
```

### 2. Spell Checker

```python
class SpellChecker:
    def __init__(self, dictionary: list):
        self.trie = Trie()
        for word in dictionary:
            self.trie.insert(word.lower())

    def is_correct(self, word: str) -> bool:
        """Check if word is spelled correctly"""
        return self.trie.search(word.lower())

    def suggest_corrections(self, word: str, max_edit_distance=2) -> list:
        """
        Suggest corrections for misspelled word
        Using edit distance (Levenshtein)
        """
        suggestions = []

        def dfs(node, current_word, remaining_edits):
            if node.is_end_of_word and current_word != word:
                suggestions.append(current_word)

            if remaining_edits == 0:
                return

            for char, child in node.children.items():
                # Try different edits
                dfs(child, current_word + char, remaining_edits - 1)

        # Simple implementation - can be optimized
        all_words = self.trie.get_all_words()
        for dict_word in all_words:
            if self._edit_distance(word, dict_word) <= max_edit_distance:
                suggestions.append(dict_word)

        return suggestions[:5]  # Top 5

    def _edit_distance(self, word1: str, word2: str) -> int:
        """Calculate Levenshtein distance"""
        m, n = len(word1), len(word2)
        dp = [[0] * (n + 1) for _ in range(m + 1)]

        for i in range(m + 1):
            dp[i][0] = i
        for j in range(n + 1):
            dp[0][j] = j

        for i in range(1, m + 1):
            for j in range(1, n + 1):
                if word1[i-1] == word2[j-1]:
                    dp[i][j] = dp[i-1][j-1]
                else:
                    dp[i][j] = 1 + min(
                        dp[i-1][j],    # Delete
                        dp[i][j-1],    # Insert
                        dp[i-1][j-1]   # Replace
                    )

        return dp[m][n]

# Usage
dictionary = ["apple", "apply", "application", "appreciate"]
checker = SpellChecker(dictionary)

print(checker.is_correct("apple"))  # True
print(checker.is_correct("aple"))   # False
print(checker.suggest_corrections("aple"))  # ['apple', 'apply']
```

### 3. IP Routing (Longest Prefix Match)

```python
class IPRouter:
    """IP routing table using trie"""

    def __init__(self):
        self.trie = Trie()
        self.routes = {}  # prefix -> next_hop

    def add_route(self, ip_prefix: str, next_hop: str):
        """
        Add routing entry
        ip_prefix: "192.168.1.0/24" -> binary prefix
        """
        binary = self._ip_to_binary(ip_prefix)
        self.trie.insert(binary)
        self.routes[binary] = next_hop

    def lookup(self, ip_address: str) -> str:
        """Find longest prefix match for IP"""
        binary = self._ip_to_binary_full(ip_address)

        # Find longest matching prefix
        best_match = None
        node = self.trie.root

        for i, bit in enumerate(binary):
            if bit not in node.children:
                break
            node = node.children[bit]
            if node.is_end_of_word:
                best_match = binary[:i+1]

        return self.routes.get(best_match, "default")

    def _ip_to_binary(self, ip_prefix: str) -> str:
        """Convert IP/mask to binary prefix"""
        ip, mask = ip_prefix.split('/')
        mask_len = int(mask)

        octets = ip.split('.')
        binary_full = ''.join(format(int(octet), '08b') for octet in octets)

        return binary_full[:mask_len]

    def _ip_to_binary_full(self, ip: str) -> str:
        """Convert IP to full binary"""
        octets = ip.split('.')
        return ''.join(format(int(octet), '08b') for octet in octets)
```

## Interview Tips

### Common Questions

**Q: Trie vs HashMap?**
- Trie: Better for prefix searches, autocomplete
- HashMap: Faster for exact lookups, simpler

**Q: Space complexity?**
- Worst case: O(ALPHABET_SIZE × N × M)
- Can optimize with compressed trie (radix tree)

**Q: When to use Trie?**
- Autocomplete
- Spell checking
- IP routing
- Dictionary/word games

**Q: Trie vs Ternary Search Tree?**
- Standard Trie: Faster, more memory
- TST: Slower, less memory

## Key Takeaways

1. **Prefix-based operations**: O(m) instead of O(n log n)
2. **Space vs Time**: Standard trie trades space for speed
3. **Compressed variants**: Radix tree, TST for space efficiency
4. **Practical uses**: Autocomplete, spell check, IP routing
5. **Implementation**: HashMap for children, boolean for word end

This data structure is essential for text processing and appears frequently in interviews!
