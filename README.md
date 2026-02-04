#include <stdio.h>
#include <string.h>
#include <ctype.h>
#include <stdlib.h>

// Convert string to lowercase
void toLower(char *str) {
    int i;
    for (i = 0; str[i]; i++) {
        str[i] = tolower(str[i]);
    }
}

// Train structure
struct Train {
    int number;
    char name[50];
    char source[20];
    char destination[20];
    int seats;
    int totalSeats;
};

// Booking structure
struct Booking {
    char passengerName[50];
    int passengerAge;
    int trainNumber;
    int seatNumber;
    char seatType[10];
};

// Seat pattern (W M A A M W) repeating
char seatPattern[6] = {'W', 'M', 'A', 'A', 'M', 'W'};

// Train data
struct Train trains[3] = {
    {101, "Chennai Express", "Chennai", "Coimbatore", 50, 50},
    {202, "Kovai Superfast", "Coimbatore", "Erode", 40, 40},
    {303, "Madurai Intercity", "Madurai", "Trichy", 60, 60}
};

int isValidName(char name[]) {
    int i;
    for (i = 0; name[i] != '\0'; i++) {
        if (!isalpha(name[i]) && name[i] != ' ')
            return 0;
    }
    return 1;
}

int isValidAge(char input[]) {
    int i;
    for (i = 0; input[i] != '\0'; i++) {
        if (!isdigit(input[i]))
            return 0;
    }
    int age = atoi(input);
    return age > 0;
}

void saveBookingToFile(struct Booking b) {
    FILE *fp = fopen("bookings.txt", "a");
    if (!fp) {
        printf("Error opening file!\n");
        return;
    }

    fprintf(fp, "%s|%d|%d|%d|%s\n",
            b.passengerName, b.passengerAge, b.trainNumber,
            b.seatNumber, b.seatType);

    fclose(fp);
}

int isDuplicatePassenger(char name[], int age) {
    FILE *fp = fopen("bookings.txt", "r");
    if (!fp)
        return 0;

    char line[200];
    char storedName[50], inputLower[50], storedLower[50];

    strcpy(inputLower, name);
    toLower(inputLower);

    while (fgets(line, sizeof(line), fp)) {
        char *pname = strtok(line, "|");
        char *page = strtok(NULL, "|");

        if (pname && page) {
            strcpy(storedName, pname);
            strcpy(storedLower, storedName);
            toLower(storedLower);

            if (strcmp(storedLower, inputLower) == 0 && atoi(page) == age) {
                fclose(fp);
                return 1;
            }
        }
    }
    fclose(fp);
    return 0;
}

void displayTrains() {
    printf("\nAvailable Trains:\n");
    printf("---------------------------------------------\n");
    int i;
    for (i = 0; i < 3; i++) {
        printf("Train No: %d\n", trains[i].number);
        printf("Name: %s\n", trains[i].name);
        printf("Route: %s -> %s\n", trains[i].source, trains[i].destination);
        printf("Seats Left: %d / %d\n", trains[i].seats, trains[i].totalSeats);
        printf("---------------------------------------------\n");
    }
}

void bookTicket() {
    int trainNo, totalPersons;
    char name[50], ageInput[10];
    int age;
    int found = -1;

    printf("\nEnter Train Number to book: ");
    scanf("%d", &trainNo);
    getchar();

    int i;
    for (i = 0; i < 3; i++) {
        if (trains[i].number == trainNo) {
            found = i;
            break;
        }
    }

    if (found == -1) {
        printf("Invalid Train Number!\n");
        return;
    }

    printf("How many persons are traveling: ");
    scanf("%d", &totalPersons);
    getchar();

    if (totalPersons <= 0 || totalPersons > trains[found].seats) {
        printf("Invalid seats / only %d available.\n", trains[found].seats);
        return;
    }

    printf("\n--- Enter Passenger Details ---\n");
    int p;
    for (p = 1; p <= totalPersons; p++) {
        struct Booking b;

        printf("\nPassenger %d Name: ", p);
        fgets(name, sizeof(name), stdin);
        name[strcspn(name, "\n")] = '\0';

        if (!isValidName(name)) {
            printf("Invalid Name!\n");
            return;
        }

        printf("Passenger %d Age: ", p);
        fgets(ageInput, sizeof(ageInput), stdin);
        ageInput[strcspn(ageInput, "\n")] = '\0';

        if (!isValidAge(ageInput)) {
            printf("Invalid Age!\n");
            return;
        }

        age = atoi(ageInput);

        if (isDuplicatePassenger(name, age)) {
            printf("Duplicate Passenger!\n");
            return;
        }

        char seatChoice;
        char type[10];

        printf("Select Seat Type (W = Window, M = Middle, A = Aisle): ");
        scanf(" %c", &seatChoice);
        getchar();

        seatChoice = toupper(seatChoice);

        if (seatChoice == 'W') strcpy(type, "Window");
        else if (seatChoice == 'M') strcpy(type, "Middle");
        else if (seatChoice == 'A') strcpy(type, "Aisle");
        else {
            printf("Invalid Seat Type!\n");
            return;
        }

        // ===== New seat allocation logic: find first free seat matching requested type =====
        int seatNumber = 0;
        int seatFound = 0;
        int s;
        for (s = 1; s <= trains[found].totalSeats; s++) {
            int idx = (s - 1) % 6;
            if (seatPattern[idx] != seatChoice) continue;

            // check if seat s on this train is already booked
            int alreadyBooked = 0;
            FILE *fp = fopen("bookings.txt", "r");
            if (fp) {
                char line[200];
                while (fgets(line, sizeof(line), fp)) {
                    char copy[200];
                    strcpy(copy, line);
                    char *pn = strtok(copy, "|");
                    char *pa = strtok(NULL, "|");
                    char *pt = strtok(NULL, "|"); // train
                    char *psn = strtok(NULL, "|"); // seat number
                    if (pt && psn) {
                        if (atoi(pt) == trainNo && atoi(psn) == s) {
                            alreadyBooked = 1;
                            break;
                        }
                    }
                }
                fclose(fp);
            }

            if (!alreadyBooked) {
                seatNumber = s;
                seatFound = 1;
                break;
            }
        }

        if (!seatFound) {
            printf("No %s seats available!\n", type);
            return;
        }

        // assign booking
        strcpy(b.passengerName, name);
        b.passengerAge = age;
        b.trainNumber = trainNo;
        b.seatNumber = seatNumber;
        strcpy(b.seatType, type);

        saveBookingToFile(b);
        trains[found].seats--;
    }

    printf("\nAll Tickets Booked Successfully!\n");
}
void cancelTicket() {
    char name[50], ageInput[10];
    int age;

    printf("\nEnter Passenger Name to Cancel: ");
    fgets(name, sizeof(name), stdin);
    name[strcspn(name, "\n")] = '\0';

    if (!isValidName(name)) {
        printf("Invalid Name!\n");
        return;
    }

    printf("Enter Passenger Age: ");
    fgets(ageInput, sizeof(ageInput), stdin);
    ageInput[strcspn(ageInput, "\n")] = '\0';

    if (!isValidAge(ageInput)) {
        printf("Invalid Age!\n");
        return;
    }

    age = atoi(ageInput);

    FILE *fp = fopen("bookings.txt", "r");
    if (!fp) {
        printf("\nNo bookings found!\n");
        return;
    }

    FILE *temp = fopen("temp.txt", "w");
    if (!temp) {
        printf("Error creating temporary file!\n");
        fclose(fp);
        return;
    }

    char line[200];
    int found = 0;

    while (fgets(line, sizeof(line), fp)) {
        char copy[200];
        strcpy(copy, line);

        char *pname = strtok(copy, "|");
        char *page = strtok(NULL, "|");
        char *ptrain = strtok(NULL, "|");
        char *pseat = strtok(NULL, "|");
        char *ptype = strtok(NULL, "|");

        if (!pname || !page || !ptrain || !pseat || !ptype)
            continue;

        // Match record?
        if (strcmp(pname, name) == 0 && atoi(page) == age) {
            found = 1;

            // return seat count to train
            int tn = atoi(ptrain);
            int i;
            for (i = 0; i < 3; i++) {
                if (trains[i].number == tn)
                    trains[i].seats++;
            }

            continue; // skip writing (deleting)
        }

        fputs(line, temp); // keep other bookings
    }

    fclose(fp);
    fclose(temp);

    remove("bookings.txt");
    rename("temp.txt", "bookings.txt");

    if (found)
        printf("\nTicket Cancelled Successfully!\n");
    else
        printf("\nPassenger Not Found!\n");
}

// showBookings (reads seat type saved in file)
void showBookings() {
    FILE *fp = fopen("bookings.txt", "r");
    if (!fp) {
        printf("\nNo bookings found!\n");
        return;
    }

    char line[200];
    struct Booking b;

    printf("\nAll Bookings:\n");
    printf("---------------------------------------------\n");

    while (fgets(line, sizeof(line), fp)) {
        char *name = strtok(line, "|");
        char *ageStr = strtok(NULL, "|");
        char *trainStr = strtok(NULL, "|");
        char *seatNum = strtok(NULL, "|");
        char *seatType = strtok(NULL, "|");

        if (!name || !ageStr || !trainStr || !seatNum || !seatType)
            continue;

        strcpy(b.passengerName, name);
        b.passengerAge = atoi(ageStr);
        b.trainNumber = atoi(trainStr);
        b.seatNumber = atoi(seatNum);

        seatType[strcspn(seatType, "\n")] = '\0';
        strcpy(b.seatType, seatType);

        printf("Passenger Name : %s\n", b.passengerName);
        printf("Age : %d\n", b.passengerAge);
        printf("Train Number : %d\n", b.trainNumber);
        printf("Seat Number : %d\n", b.seatNumber);
        printf("Seat Type : %s\n", b.seatType);
        printf("---------------------------------------------\n");
    }
    fclose(fp);
}

int main() {
    FILE *fp = fopen("bookings.txt", "w");
    if (fp) fclose(fp);

    char input[10];
    int choice;

    while (1) {
        printf("\n===== Railway Ticket Booking System =====\n");
        printf("1. Display Trains\n");
        printf("2. Book Ticket\n");
        printf("3. Show Bookings\n");
        printf("4. Cancel Ticket\n");
        printf("5. Exit\n");
        printf("Enter your choice: ");

        fgets(input, sizeof(input), stdin);
        if (!isdigit(input[0])) {
            printf("\nInvalid input!\n");
            continue;
        }

        choice = atoi(input);

        switch (choice) {
        case 1:
            displayTrains();
            break;
        case 2:
            bookTicket();
            break;
        case 3:
            showBookings();
            break;
        case 4:
            cancelTicket();
            break;
        case 5:
            printf("Thank you!\n");
            return 0;
        default:
            printf("Invalid choice!\n");
        }
    }
}

