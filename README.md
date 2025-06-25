# Freelance-Market-Place-Contract
💼 FreelanceMarketplace Smart Contract – Description This smart contract implements a decentralized freelance job marketplace on the Ethereum blockchain. It allows clients to post jobs and freelancers to bid, work, and receive milestone-based payments securely.

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract FreelanceMarketplace {

    address public owner;

    uint public jobCounter = 1;

    constructor() {
        owner = msg.sender;
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "Not contract owner");
        _;
    }

    enum JobStatus { Open, Assigned, Completed, Cancelled }

    struct User {
        bool isRegistered;
        string name;
        bool isFreelancer;
        uint totalJobs;
        uint ratingSum;
    }

    struct Job {
        uint id;
        address client;
        string title;
        string description;
        uint budget;
        JobStatus status;
        address freelancer;
        uint milestoneAmount;
        uint milestonesCompleted;
        uint totalMilestones;
        uint creationTime;
    }

    mapping(address => User) public users;
    mapping(uint => Job) public jobs;
    mapping(uint => address[]) public jobBidders;
    mapping(address => uint[]) public jobsByUser;

    event UserRegistered(address indexed user, string name, bool isFreelancer);
    event JobPosted(uint indexed jobId, address indexed client);
    event BidPlaced(uint indexed jobId, address indexed freelancer);
    event FreelancerAssigned(uint indexed jobId, address indexed freelancer);
    event MilestoneCompleted(uint indexed jobId, uint milestoneNo);
    event JobCompleted(uint indexed jobId);
    event RatingGiven(address indexed user, uint rating);
    event FundsTransferred(address indexed to, uint amount);

    // Register as a client or freelancer
    function register(string memory name, bool isFreelancer) external {
        require(!users[msg.sender].isRegistered, "Already registered");
        users[msg.sender] = User(true, name, isFreelancer, 0, 0);
        emit UserRegistered(msg.sender, name, isFreelancer);
    }

    // Post a new job
    function postJob(string memory title, string memory description, uint budget, uint totalMilestones) external {
        require(users[msg.sender].isRegistered, "Not registered");
        require(!users[msg.sender].isFreelancer, "Freelancers can't post jobs");

        uint jobId = jobCounter++;
        jobs[jobId] = Job(
            jobId,
            msg.sender,
            title,
            description,
            budget,
            JobStatus.Open,
            address(0),
            budget / totalMilestones,
            0,
            totalMilestones,
            block.timestamp
        );
        jobsByUser[msg.sender].push(jobId);
        emit JobPosted(jobId, msg.sender);
    }

    // Freelancers place bid on job
    function placeBid(uint jobId) external {
        require(users[msg.sender].isRegistered && users[msg.sender].isFreelancer, "Must be a registered freelancer");
        Job storage job = jobs[jobId];
        require(job.status == JobStatus.Open, "Job not open for bidding");

        jobBidders[jobId].push(msg.sender);
        emit BidPlaced(jobId, msg.sender);
    }

    // Client assigns job to a freelancer
    function assignFreelancer(uint jobId, address freelancer) external payable {
        Job storage job = jobs[jobId];
        require(msg.sender == job.client, "Only client can assign");
        require(job.status == JobStatus.Open, "Job already assigned or closed");
        require(msg.value == job.budget, "Must fund full job amount");

        job.freelancer = freelancer;
        job.status = JobStatus.Assigned;
        emit FreelancerAssigned(jobId, freelancer);
    }

    // Freelancer completes milestone
    function completeMilestone(uint jobId) external {
        Job storage job = jobs[jobId];
        require(msg.sender == job.freelancer, "Not assigned freelancer");
        require(job.status == JobStatus.Assigned, "Job not active");
        require(job.milestonesCompleted < job.totalMilestones, "All milestones already completed");

        job.milestonesCompleted++;
        payable(msg.sender).transfer(job.milestoneAmount);
        emit MilestoneCompleted(jobId, job.milestonesCompleted);
        emit FundsTransferred(msg.sender, job.milestoneAmount);

        if (job.milestonesCompleted == job.totalMilestones) {
            job.status = JobStatus.Completed;
            emit JobCompleted(jobId);
        }
    }

    // Give rating to freelancer or client
    function rateUser(address userAddr, uint rating) external {
        require(users[userAddr].isRegistered, "User not found");
        require(rating > 0 && rating <= 5, "Invalid rating");

        users[userAddr].ratingSum += rating;
        users[userAddr].totalJobs++;
        emit RatingGiven(userAddr, rating);
    }

    // Get average rating of a user
    function getUserRating(address userAddr) public view returns (uint) {
        User memory user = users[userAddr];
        if (user.totalJobs == 0) return 0;
        return user.ratingSum / user.totalJobs;
    }

    // Get job details by user
    function getJobsByUser(address userAddr) external view returns (uint[] memory) {
        return jobsByUser[userAddr];
    }

    // Cancel job (only if not assigned)
    function cancelJob(uint jobId) external {
        Job storage job = jobs[jobId];
        require(msg.sender == job.client, "Only client can cancel");
        require(job.status == JobStatus.Open, "Cannot cancel assigned job");

        job.status = JobStatus.Cancelled;
    }

    // Emergency withdrawal by owner (e.g. if funds stuck)
    function emergencyWithdraw() external onlyOwner {
        uint bal = address(this).balance;
        payable(owner).transfer(bal);
    }

    // Fallback to receive ETH
    receive() external payable {}
}
