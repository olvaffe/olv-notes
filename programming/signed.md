signed pros:

int size = sizeof(entry) * n_entries;
if (size <= 0) { /* this check is not abnormal */
	return;
}

unsigned size = sizeof(entry) * n_entries;
if (size <= n_entries) { /* this check is abnormal */
	return;
}
