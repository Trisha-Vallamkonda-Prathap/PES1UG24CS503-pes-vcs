# PES-VCS Submission Report

**Name:** Trisha Vallamkonda Prathap  
**SRN:** PES1UG24CS503  
**Section:** I

---

## Submission Checklist

- Repository contains all required source files (`object.c`, `tree.c`, `index.c`, `commit.c`, headers, tests, `Makefile`)
- Screenshot evidence provided for Phases 1 through 5
- Phase 5 analysis questions answered (Q5.1, Q5.2, Q5.3)
- Phase 6 analysis questions answered (Q6.1, Q6.2)
- PDF report available at repository root: `report.pdf`

---
#All screenshots are provided in report.pdf

## Phase 5 Questions and Answers

### Q5.1

**Question:** A branch in Git is just a file in `.git/refs/heads/` containing a commit hash. Creating a branch is creating a file. Given this, how would you implement `pes checkout <branch>` — what files need to change in `.pes/`, and what must happen to the working directory? What makes this operation complex?

**Answer:**

To implement `pes checkout <branch>`, first read the commit hash stored in `.pes/refs/heads/<branch>`. Using that hash, load the commit object and identify the tree linked to it. Then update `HEAD` so that it points to the selected branch reference. After that, rebuild the index according to the target branch snapshot. The working directory must also be changed so it matches that tree exactly — by adding new files, updating modified ones, and removing tracked files that are not present in the target branch.

The difficult part is that checkout is more than changing one reference. It has to safely sync branch data, index data, and user files while making sure local uncommitted changes are not lost and the repository does not end up partially updated.

---

### Q5.2

**Question:** When switching branches, the working directory must be updated to match the target branch's tree. If the user has uncommitted changes to a tracked file, and that file differs between branches, checkout must refuse. Describe how you would detect this "dirty working directory" conflict using only the index and the object store.

**Answer:**

This conflict can be detected by comparing the current working directory with the index, and then comparing those results with the target branch tree stored in the object database. For every tracked file in the index, first use metadata such as size or modification time for a quick check, and if needed calculate the file hash. If the current file hash does not match the index hash, then the file has local changes.

Next, check whether that same path is different in the target branch tree — such as a changed blob, deleted file, or type difference. If both conditions are true, checkout should stop because switching branches would overwrite local work. It should also refuse if an untracked file exists where the target branch needs to create a file.

---

### Q5.3

**Question:** "Detached HEAD" means HEAD contains a commit hash directly instead of a branch reference. What happens if you make commits in this state? How could a user recover those commits?

**Answer:**

In detached HEAD state, `HEAD` stores a commit hash directly instead of pointing to a branch. If commits are made in this state, new commits are still created normally, but no branch pointer moves forward to include them. Because of this, those commits can become hard to find once the user checks out another branch.

To recover them, the user can create a new branch that points to the detached commit hash, then switch to that branch. As long as garbage collection has not removed those unreachable commits, they can still be restored safely.

---

## Phase 6 Questions and Answers

### Q6.1

**Question:** Over time, the object store accumulates unreachable objects — blobs, trees, or commits that no branch points to (directly or transitively). Describe an algorithm to find and delete these objects. What data structure would you use to track "reachable" hashes efficiently? For a repository with 100,000 commits and 50 branches, estimate how many objects you'd need to visit.

**Answer:**

A safe way to clean unreachable objects is to use a mark-and-sweep method. In the marking phase, begin from all active references such as files in `.pes/refs/heads/*` and also `HEAD` if it is detached. From these starting points, recursively follow commit links to parent commits and trees, then continue through each tree to the blobs and subtrees it contains.

Use a hash set to store every visited hash, since membership checks are fast and each object is processed only once even if multiple branches share history. In the sweep phase, scan all files inside `.pes/objects/*`. If an object hash is not found in the reachable set, it can be deleted.

For a repository with 100,000 commits and 50 branches, the total visited objects would usually be close to the number of unique commits plus their related trees and blobs. Since branches often share history, it is normally much less than 50 times the commit count, and may range from a few hundred thousand to a few million objects depending on repository contents.

---

### Q6.2

**Question:** Why is it dangerous to run garbage collection concurrently with a commit operation? Describe a race condition where GC could delete an object that a concurrent commit is about to reference. How does Git's real GC avoid this?

**Answer:**

Running garbage collection at the same time as a commit is risky because creating a commit happens in multiple steps. Usually blobs, trees, and the commit object are written first, and only after that does the branch reference get updated.

A race condition can happen if GC scans references before the branch is updated. It may not see the newly created commit as reachable and could delete one of the new objects. Then, when the branch ref is finally updated, it may point to a commit whose required objects are missing.

Git avoids this by using locking during updates, keeping recently unreachable objects for a grace period instead of deleting them immediately, and using reflogs for extra protection. This ensures new objects are not removed while a commit operation is still in progress.

---

## Final Notes

- Minimum 5 meaningful commits per phase maintained in Git history
- `report.pdf` is available at the repository root
- All source files (`object.c`, `tree.c`, `index.c`, `commit.c`) are implemented and tested
